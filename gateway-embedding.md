## gateway-embedding

```bash
curl -sS -o /tmp/embed_resp.json \
  -w '\nconnect=%{time_connect}s\nttfb=%{time_starttransfer}s\ne2e=%{time_total}s\n' \
  -X POST http://192.168.86.179:30181/v1/embeddings \
  -H "X-Request-Id: request_id_1" \
  -H "X-Trace-Id: trace_id_1" \
  -H "X-Session-Id: session_id_1" \
  -H "Content-Type: application/json" \
  -d '{"model":"BAAI/bge-m3","input":"hello world"}'
```
#### test tokens -> max-model-len

```bash
cat >/tmp/bench_embed.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

percentile_99() {
  sort -n | awk '
    { a[NR] = $1 }
    END {
      if (NR == 0) { print "NA"; exit }
      idx = int(NR * 0.99)
      if (idx < 1) idx = 1
      if (idx > NR) idx = NR
      print a[idx]
    }'
}

BACKENDS=(
  "http://192.168.86.173:8001"
  "http://192.168.86.176:8001"
  "http://192.168.86.179:30181"
)

SIZES=(5000 10000 13000 15000 30000)

SOURCE_URL="https://en.wikipedia.org/wiki/New_York_City"
INPUT_FILE=/tmp/vllm_embed_input.txt
PAYLOAD=/tmp/vllm_embed.json

TOTAL=100
CONCURRENCY=20

echo "Fetching source..."
RAW=/tmp/wiki_raw.txt
curl -fsSL "$SOURCE_URL" \
  | lynx -dump -stdin \
  | iconv -f utf-8 -t utf-8 -c > "$RAW"

echo ""
echo "NOTE:"
echo "- 192.168.86.173:8001 and 192.168.86.176:8001 are direct backend tests."
echo "- 192.168.86.179:30181 is the gateway test."
echo "- Gateway results are NOT perfectly apples-to-apples with direct backend results."
echo "- The gateway path may add routing / queue / proxy overhead."
echo "- In your single curl example, the gateway request also includes X-Request-Id / X-Trace-Id / X-Session-Id headers."

for SIZE in "${SIZES[@]}"; do
  echo ""
  echo "================ SIZE=${SIZE} ================="

  head -c "$SIZE" "$RAW" > "$INPUT_FILE"

  raw_chars=$(wc -c < "$INPUT_FILE" | tr -d ' ')
  tokens=$(( raw_chars / 4 ))

  python3 - <<PY
import json

with open("$INPUT_FILE", "r", encoding="utf-8", errors="ignore") as f:
    text = f.read()

payload = {"model": "BAAI/bge-m3", "input": text}

with open("$PAYLOAD", "w", encoding="utf-8") as out:
    json.dump(payload, out)
PY

  for ENDPOINT in "${BACKENDS[@]}"; do
    tmpfile=$(mktemp)

    if [[ "$ENDPOINT" == "http://192.168.86.179:30181" ]]; then
      target_type="gateway"
      seq 1 $TOTAL | xargs -P $CONCURRENCY -I{} bash -c '
        i="$1"
        endpoint="$2"
        payload="$3"

        curl -sS -o /dev/null \
          -w "%{http_code} %{time_connect} %{time_starttransfer} %{time_total}\n" \
          -X POST "$endpoint/v1/embeddings" \
          -H "X-Request-Id: request_id_$i" \
          -H "X-Trace-Id: trace_id_$i" \
          -H "X-Session-Id: session_id_$i" \
          -H "Content-Type: application/json" \
          --data-binary @"$payload"
      ' _ {} "$ENDPOINT" "$PAYLOAD" > "$tmpfile"
    else
      target_type="direct"
      seq 1 $TOTAL | xargs -P $CONCURRENCY -I{} bash -c '
        endpoint="$1"
        payload="$2"

        curl -sS -o /dev/null \
          -w "%{http_code} %{time_connect} %{time_starttransfer} %{time_total}\n" \
          -X POST "$endpoint/v1/embeddings" \
          -H "Content-Type: application/json" \
          --data-binary @"$payload"
      ' _ "$ENDPOINT" "$PAYLOAD" > "$tmpfile"
    fi

    success=$(awk '$1 == 200 {c++} END {print c+0}' "$tmpfile")
    total=$(wc -l < "$tmpfile" | tr -d ' ')
    errors=$(( total - success ))

    p99_connect=$(awk '$1 == 200 {print $2}' "$tmpfile" | percentile_99)
    p99_ttfb=$(awk '$1 == 200 {print $3}' "$tmpfile" | percentile_99)
    p99_e2e=$(awk '$1 == 200 {print $4}' "$tmpfile" | percentile_99)

    echo "backend=$ENDPOINT type=$target_type input_chars=$raw_chars approx_tokens=$tokens total=$total success=$success errors=$errors p99_connect=${p99_connect}s p99_ttfb=${p99_ttfb}s p99_e2e=${p99_e2e}s"

    rm -f "$tmpfile"
  done
done

rm -f "$INPUT_FILE" "$PAYLOAD" "$RAW"
echo "Done."
EOF

bash /tmp/bench_embed.sh
```

Sample output (reference):

```
================ SIZE=5000 =================
backend=http://192.168.86.173:8001 type=direct input_chars=5000 approx_tokens=1250 total=100 success=100 errors=0 p99_connect=0.021955s p99_ttfb=0.409331s p99_e2e=0.420935s
backend=http://192.168.86.176:8001 type=direct input_chars=5000 approx_tokens=1250 total=100 success=100 errors=0 p99_connect=0.017475s p99_ttfb=0.398404s p99_e2e=0.411136s
backend=http://192.168.86.179:30181 type=gateway input_chars=5000 approx_tokens=1250 total=100 success=100 errors=0 p99_connect=0.049222s p99_ttfb=0.426182s p99_e2e=0.454256s

================ SIZE=10000 =================
backend=http://192.168.86.173:8001 type=direct input_chars=10000 approx_tokens=2500 total=100 success=100 errors=0 p99_connect=0.017799s p99_ttfb=0.963546s p99_e2e=0.973953s
backend=http://192.168.86.176:8001 type=direct input_chars=10000 approx_tokens=2500 total=100 success=100 errors=0 p99_connect=0.047261s p99_ttfb=0.960085s p99_e2e=0.971856s
backend=http://192.168.86.179:30181 type=gateway input_chars=10000 approx_tokens=2500 total=100 success=100 errors=0 p99_connect=0.043642s p99_ttfb=0.938038s p99_e2e=0.974017s

================ SIZE=13000 =================
backend=http://192.168.86.173:8001 type=direct input_chars=13000 approx_tokens=3250 total=100 success=0 errors=100 p99_connect=NAs p99_ttfb=NAs p99_e2e=NAs
backend=http://192.168.86.176:8001 type=direct input_chars=13000 approx_tokens=3250 total=100 success=0 errors=100 p99_connect=NAs p99_ttfb=NAs p99_e2e=NAs
backend=http://192.168.86.179:30181 type=gateway input_chars=13000 approx_tokens=3250 total=100 success=0 errors=100 p99_connect=NAs p99_ttfb=NAs p99_e2e=NAs

================ SIZE=15000 =================
backend=http://192.168.86.173:8001 type=direct input_chars=15000 approx_tokens=3750 total=100 success=0 errors=100 p99_connect=NAs p99_ttfb=NAs p99_e2e=NAs
backend=http://192.168.86.176:8001 type=direct input_chars=15000 approx_tokens=3750 total=100 success=0 errors=100 p99_connect=NAs p99_ttfb=NAs p99_e2e=NAs
backend=http://192.168.86.179:30181 type=gateway input_chars=15000 approx_tokens=3750 total=100 success=0 errors=100 p99_connect=NAs p99_ttfb=NAs p99_e2e=NAs

================ SIZE=30000 =================
backend=http://192.168.86.173:8001 type=direct input_chars=30000 approx_tokens=7500 total=100 success=0 errors=100 p99_connect=NAs p99_ttfb=NAs p99_e2e=NAs
backend=http://192.168.86.176:8001 type=direct input_chars=30000 approx_tokens=7500 total=100 success=0 errors=100 p99_connect=NAs p99_ttfb=NAs p99_e2e=NAs
backend=http://192.168.86.179:30181 type=gateway input_chars=30000 approx_tokens=7500 total=100 success=0 errors=100 p99_connect=NAs p99_ttfb=NAs p99_e2e=NAs
```

#### test concurrent with small tokens -> max-num-seqs
```
percentile_99() {
  sort -n | awk '
    { a[NR] = $1 }
    END {
      if (NR == 0) { print "NA"; exit }
      idx = int(NR * 0.99)
      if (idx < 1) idx = 1
      if (idx > NR) idx = NR
      print a[idx]
    }'
}

BACKENDS=(
  "http://192.168.86.173:8001"
  "http://192.168.86.176:8001"
  "http://192.168.86.179:30181"
)

SOURCE_URL="https://en.wikipedia.org/wiki/New_York_City"
TOTAL_REQUESTS=500
CONCURRENCY=80
INPUT_CHARS=300

INPUT_FILE=/tmp/vllm_embed_input.txt
PAYLOAD=/tmp/vllm_embed.json

echo "NOTE:"
echo "- 192.168.86.173:8001 and 192.168.86.176:8001 are direct backend tests."
echo "- 192.168.86.179:30181 is the gateway test."
echo "- Gateway results are not perfectly apples-to-apples with direct backend results."
echo "- The JSON shape is the same, but the gateway path adds proxy/routing overhead."
echo "- The gateway requests below also include X-Request-Id / X-Trace-Id / X-Session-Id headers."

curl -fsSL "$SOURCE_URL" \
  | lynx -dump -stdin \
  | iconv -f utf-8 -t utf-8 -c \
  | head -c "$INPUT_CHARS" > "$INPUT_FILE"

raw_chars=$(wc -c <"$INPUT_FILE" | tr -d ' ')
approx_tokens=$(( raw_chars / 4 ))

python3 - <<'PY'
import json
with open("/tmp/vllm_embed_input.txt", "r", encoding="utf-8", errors="ignore") as f:
    payload = {"model": "BAAI/bge-m3", "input": f.read()}
with open("/tmp/vllm_embed.json", "w", encoding="utf-8") as out:
    json.dump(payload, out)
PY

for ENDPOINT in "${BACKENDS[@]}"; do
  tmpfile=$(mktemp)

  if [[ "$ENDPOINT" == "http://192.168.86.179:30181" ]]; then
    target_type="gateway"

    seq 1 "$TOTAL_REQUESTS" | xargs -P "$CONCURRENCY" -I{} bash -c '
      i="$1"
      endpoint="$2"
      payload="$3"

      curl -sS -o /dev/null \
        -w "%{http_code} %{time_connect} %{time_starttransfer} %{time_total}\n" \
        -X POST "$endpoint/v1/embeddings" \
        -H "X-Request-Id: request_id_$i" \
        -H "X-Trace-Id: trace_id_$i" \
        -H "X-Session-Id: session_id_$i" \
        -H "Content-Type: application/json" \
        --data-binary @"$payload"
    ' _ {} "$ENDPOINT" "$PAYLOAD" >"$tmpfile"
  else
    target_type="direct"

    seq 1 "$TOTAL_REQUESTS" | xargs -P "$CONCURRENCY" -I{} bash -c '
      endpoint="$1"
      payload="$2"

      curl -sS -o /dev/null \
        -w "%{http_code} %{time_connect} %{time_starttransfer} %{time_total}\n" \
        -X POST "$endpoint/v1/embeddings" \
        -H "Content-Type: application/json" \
        --data-binary @"$payload"
    ' _ "$ENDPOINT" "$PAYLOAD" >"$tmpfile"
  fi

  success=$(awk '$1 == "200" { c++ } END { print c + 0 }' "$tmpfile")
  total=$(wc -l <"$tmpfile" | tr -d " ")
  errors=$((total - success))

  p99_connect=$(awk '$1 == "200" { print $2 }' "$tmpfile" | percentile_99)
  p99_ttfb=$(awk '$1 == "200" { print $3 }' "$tmpfile" | percentile_99)
  p99_e2e=$(awk '$1 == "200" { print $4 }' "$tmpfile" | percentile_99)

  echo "backend=$ENDPOINT type=$target_type input_chars=$raw_chars approx_tokens=$approx_tokens total=$total success=$success errors=$errors p99_connect=${p99_connect}s p99_ttfb=${p99_ttfb}s p99_e2e=${p99_e2e}s"

  rm -f "$tmpfile"
done

rm -f "$INPUT_FILE" "$PAYLOAD"
```

```
backend=http://192.168.86.173:8001 type=direct input_chars=300 approx_tokens=75 total=500 success=500 errors=0 p99_connect=0.050657s p99_ttfb=0.110533s p99_e2e=0.243068s
backend=http://192.168.86.176:8001 type=direct input_chars=300 approx_tokens=75 total=500 success=500 errors=0 p99_connect=0.141611s p99_ttfb=0.276727s p99_e2e=0.715379s
backend=http://192.168.86.179:30181 type=gateway input_chars=300 approx_tokens=75 total=500 success=500 errors=0 p99_connect=0.308159s p99_ttfb=0.630069s p99_e2e=1.019490s
```





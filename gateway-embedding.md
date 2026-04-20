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
#### ================ SMALL TEST ================

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
================ SMALL TEST ================
concurrency=20
http://192.168.86.173:8001 total=200 success=200 errors=0 p99_ttfb=0.174752s p99_e2e=0.187715s
http://192.168.86.176:8001 total=200 success=200 errors=0 p99_ttfb=0.540942s p99_e2e=0.629698s
http://192.168.86.179:30181 total=200 success=0 errors=200 p99_ttfb=NAs p99_e2e=NAs

concurrency=60
http://192.168.86.173:8001 total=200 success=200 errors=0 p99_ttfb=0.313707s p99_e2e=0.327615s
http://192.168.86.176:8001 total=200 success=200 errors=0 p99_ttfb=0.318511s p99_e2e=0.351055s
http://192.168.86.179:30181 total=200 success=0 errors=200 p99_ttfb=NAs p99_e2e=NAs

concurrency=100
http://192.168.86.173:8001 total=200 success=200 errors=0 p99_ttfb=0.525701s p99_e2e=0.542246s
http://192.168.86.176:8001 total=200 success=200 errors=0 p99_ttfb=0.515646s p99_e2e=0.529888s
http://192.168.86.179:30181 total=200 success=0 errors=200 p99_ttfb=NAs p99_e2e=NAs

concurrency=140
http://192.168.86.173:8001 total=200 success=200 errors=0 p99_ttfb=0.654717s p99_e2e=0.665506s
http://192.168.86.176:8001 total=200 success=200 errors=0 p99_ttfb=0.586319s p99_e2e=0.597465s
http://192.168.86.179:30181 total=200 success=0 errors=200 p99_ttfb=NAs p99_e2e=NAs

concurrency=180
http://192.168.86.173:8001 total=200 success=200 errors=0 p99_ttfb=0.750949s p99_e2e=0.763222s
http://192.168.86.176:8001 total=200 success=200 errors=0 p99_ttfb=0.613822s p99_e2e=0.625734s
http://192.168.86.179:30181 total=200 success=0 errors=200 p99_ttfb=NAs p99_e2e=NAs
```

#### ================ LARGE TEST ================
```
percentile_99() {
  sort -n | awk '
    { a[NR] = $1 }
    END {
      if (NR == 0) { print "NA"; exit }
      idx = int(NR * 0.99)
      if (idx < 1) idx = 1
      print a[idx]
    }'
}

BACKENDS=(
  "http://192.168.86.173:8001"
  "http://192.168.86.176:8001"
  "http://192.168.86.179:30181"
)

SOURCE_URL="https://en.wikipedia.org/wiki/New_York_City"
TOTAL_REQUESTS=200
CONCURRENCY_LEVELS=(20 60 100 140 180)

INPUT_CHARS=8000

RAW_TEXT=/tmp/wiki_raw.txt
TEXT=/tmp/wiki_large.txt
PAYLOAD_FILE=/tmp/embed_large_payload.json

# prepare text
curl -fsSL "$SOURCE_URL" | lynx -dump -stdin | iconv -f utf-8 -t utf-8 -c >"$RAW_TEXT"
head -c "$INPUT_CHARS" "$RAW_TEXT" >"$TEXT"

# build payload
python3 - <<'PY'
import json
with open("/tmp/wiki_large.txt", "r", encoding="utf-8", errors="ignore") as f:
    payload = {"model": "BAAI/bge-m3", "input": f.read()}
with open("/tmp/embed_large_payload.json", "w", encoding="utf-8") as out:
    json.dump(payload, out)
PY

echo "================ LARGE TEST ================"

for P in "${CONCURRENCY_LEVELS[@]}"; do
  echo "concurrency=$P"

  for ENDPOINT in "${BACKENDS[@]}"; do
    tmpfile=$(mktemp)

    seq 1 "$TOTAL_REQUESTS" | xargs -P "$P" -I{} bash -c '
      curl -sS -o /dev/null \
        -w "%{http_code} %{time_starttransfer} %{time_total}\n" \
        -X POST "$1/v1/embeddings" \
        -H "Content-Type: application/json" \
        --data-binary @"$2"
    ' _ "$ENDPOINT" "$PAYLOAD_FILE" >"$tmpfile" 2>/dev/null

    success=$(awk '$1 == "200" { c++ } END { print c + 0 }' "$tmpfile")
    errors=$((TOTAL_REQUESTS - success))

    p99_ttfb=$(awk '$1 == "200" { print $2 }' "$tmpfile" | percentile_99)
    p99_e2e=$(awk '$1 == "200" { print $3 }' "$tmpfile" | percentile_99)

    echo "$ENDPOINT total=$TOTAL_REQUESTS success=$success errors=$errors p99_ttfb=${p99_ttfb}s p99_e2e=${p99_e2e}s"

    rm -f "$tmpfile"
  done

  echo
done
```

```
concurrency=20
http://192.168.86.173:8001 total=200 success=200 errors=0 p99_ttfb=0.730986s p99_e2e=0.740059s
http://192.168.86.176:8001 total=200 success=200 errors=0 p99_ttfb=0.671520s p99_e2e=0.683831s
http://192.168.86.179:30181 total=200 success=0 errors=200 p99_ttfb=NAs p99_e2e=NAs

concurrency=60
http://192.168.86.173:8001 total=200 success=200 errors=0 p99_ttfb=2.093918s p99_e2e=2.108102s
http://192.168.86.176:8001 total=200 success=200 errors=0 p99_ttfb=2.060932s p99_e2e=2.070387s
http://192.168.86.179:30181 total=200 success=0 errors=200 p99_ttfb=NAs p99_e2e=NAs

concurrency=100
http://192.168.86.173:8001 total=200 success=200 errors=0 p99_ttfb=3.512805s p99_e2e=3.523750s
http://192.168.86.176:8001 total=200 success=200 errors=0 p99_ttfb=4.368794s p99_e2e=4.380180s
http://192.168.86.179:30181 total=200 success=0 errors=200 p99_ttfb=NAs p99_e2e=NAs

concurrency=140
http://192.168.86.173:8001 total=200 success=200 errors=0 p99_ttfb=4.926736s p99_e2e=4.939784s
http://192.168.86.176:8001 total=200 success=200 errors=0 p99_ttfb=4.861994s p99_e2e=4.890981s
http://192.168.86.179:30181 total=200 success=0 errors=200 p99_ttfb=NAs p99_e2e=NAs

concurrency=180
http://192.168.86.173:8001 total=200 success=200 errors=0 p99_ttfb=6.354471s p99_e2e=6.364525s
http://192.168.86.176:8001 total=200 success=200 errors=0 p99_ttfb=6.244042s p99_e2e=6.254960s
http://192.168.86.179:30181 total=200 success=0 errors=200 p99_ttfb=NAs p99_e2e=NAs
```





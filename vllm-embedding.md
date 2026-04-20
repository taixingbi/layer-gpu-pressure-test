# vLLM embedding (direct nodes)

## Single-request smoke

gpu-node-1:

```bash
curl -sS -o /tmp/embed_resp.json \
  -w '\nconnect=%{time_connect}s\nttfb=%{time_starttransfer}s\ne2e=%{time_total}s\n' \
  -X POST http://192.168.86.173:8001/v1/embeddings \
  -H "Content-Type: application/json" \
  -d '{"model":"BAAI/bge-m3","input":"hello world"}'
```

Example timing:

```
connect=0.056271s
ttfb=0.102769s
e2e=0.113283s
```

gpu-node-2:

```bash
curl -sS -o /tmp/embed_resp.json \
  -w '\nconnect=%{time_connect}s\nttfb=%{time_starttransfer}s\ne2e=%{time_total}s\n' \
  -X POST http://192.168.86.176:8001/v1/embeddings \
  -H "X-Request-Id: request_id_1" \
  -H "X-Trace-Id: trace_id_1" \
  -H "X-Session-Id: session_id_1" \
  -H "Content-Type: application/json" \
  -d '{"model":"BAAI/bge-m3","input":"hello world"}'
```

Example timing:

```
connect=0.048019s
ttfb=0.090159s
e2e=0.106381s
```

## test tokens -> max-model-len

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

    seq 1 $TOTAL | xargs -P $CONCURRENCY -I{} bash -c '
      curl -sS -o /dev/null \
        -w "%{http_code} %{time_starttransfer} %{time_total}\n" \
        -X POST "$1/v1/embeddings" \
        -H "Content-Type: application/json" \
        --data-binary @"$2"
    ' _ "$ENDPOINT" "$PAYLOAD" > "$tmpfile"

    success=$(awk '$1 == 200 {c++} END {print c+0}' "$tmpfile")
    total=$(wc -l < "$tmpfile" | tr -d ' ')
    errors=$(( total - success ))

    p99_ttfb=$(awk '$1 == 200 {print $2}' "$tmpfile" | percentile_99)
    p99_e2e=$(awk '$1 == 200 {print $3}' "$tmpfile" | percentile_99)

    echo "backend=$ENDPOINT input_chars=$raw_chars approx_tokens=$tokens total=$total success=$success errors=$errors p99_ttfb=${p99_ttfb}s p99_e2e=${p99_e2e}s"

    rm -f "$tmpfile"
  done
done

rm -f "$INPUT_FILE" "$PAYLOAD" "$RAW"
echo "Done."
EOF

bash /tmp/bench_embed.sh
```

Sample output:

```
================ SIZE=5000 =================
backend=http://192.168.86.173:8001 input_chars=5000 approx_tokens=1250 total=100 success=100 errors=0 p99_ttfb=0.413656s p99_e2e=0.430946s
backend=http://192.168.86.176:8001 input_chars=5000 approx_tokens=1250 total=100 success=100 errors=0 p99_ttfb=0.389468s p99_e2e=0.401716s

================ SIZE=10000 =================
backend=http://192.168.86.173:8001 input_chars=10000 approx_tokens=2500 total=100 success=100 errors=0 p99_ttfb=1.031882s p99_e2e=1.043016s
backend=http://192.168.86.176:8001 input_chars=10000 approx_tokens=2500 total=100 success=100 errors=0 p99_ttfb=0.962121s p99_e2e=0.971111s

================ SIZE=13000 =================
backend=http://192.168.86.173:8001 input_chars=13000 approx_tokens=3250 total=100 success=0 errors=100 p99_ttfb=NAs p99_e2e=NAs
backend=http://192.168.86.176:8001 input_chars=13000 approx_tokens=3250 total=100 success=0 errors=100 p99_ttfb=NAs p99_e2e=NAs

================ SIZE=15000 =================
backend=http://192.168.86.173:8001 input_chars=15000 approx_tokens=3750 total=100 success=0 errors=100 p99_ttfb=NAs p99_e2e=NAs
backend=http://192.168.86.176:8001 input_chars=15000 approx_tokens=3750 total=100 success=0 errors=100 p99_ttfb=NAs p99_e2e=NAs

================ SIZE=30000 =================
backend=http://192.168.86.173:8001 input_chars=30000 approx_tokens=7500 total=100 success=0 errors=100 p99_ttfb=NAs p99_e2e=NAs
backend=http://192.168.86.176:8001 input_chars=30000 approx_tokens=7500 total=100 success=0 errors=100 p99_ttfb=NAs p99_e2e=NAs
```

## test concurrent with large tokens ->  max-num-seqs

```bash
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
)

SOURCE_URL="https://en.wikipedia.org/wiki/New_York_City"
TOTAL_REQUESTS=500
CONCURRENCY=80
INPUT_CHARS=8000

INPUT_FILE=/tmp/vllm_embed_input.txt
LARGE_PAYLOAD=/tmp/vllm_embed_large.json

curl -fsSL "$SOURCE_URL" | lynx -dump -stdin | iconv -f utf-8 -t utf-8 -c | head -c "$INPUT_CHARS" >"$INPUT_FILE"

raw_chars=$(wc -c <"$INPUT_FILE" | tr -d ' ')
approx_tokens=$(( raw_chars / 4 ))

echo "input_chars=$raw_chars approx_tokens=$approx_tokens"

python3 - <<'PY'
import json

with open("/tmp/vllm_embed_input.txt", "r", encoding="utf-8", errors="ignore") as f:
    payload = {"model": "BAAI/bge-m3", "input": f.read()}
with open("/tmp/vllm_embed_large.json", "w", encoding="utf-8") as out:
    json.dump(payload, out)
PY

for ENDPOINT in "${BACKENDS[@]}"; do
  tmpfile=$(mktemp)
  seq 1 "$TOTAL_REQUESTS" | xargs -P "$CONCURRENCY" -I{} bash -c '
    curl -sS -o /dev/null \
      -w "%{http_code} %{time_starttransfer} %{time_total}\n" \
      -X POST "$1/v1/embeddings" \
      -H "Content-Type: application/json" \
      --data-binary @'"$LARGE_PAYLOAD"'
  ' _ "$ENDPOINT" >"$tmpfile"

  success=$(awk '$1 == "200" { c++ } END { print c + 0 }' "$tmpfile")
  total=$(wc -l <"$tmpfile" | tr -d " ")
  errors=$((total - success))

  p99_ttfb=$(awk '$1 == "200" { print $2 }' "$tmpfile" | percentile_99)
  p99_e2e=$(awk '$1 == "200" { print $3 }' "$tmpfile" | percentile_99)

  echo "backend=$ENDPOINT input_chars=$raw_chars approx_tokens=$approx_tokens total=$total success=$success errors=$errors p99_ttfb=${p99_ttfb}s p99_e2e=${p99_e2e}s"
  rm -f "$tmpfile"
done
```

Sample output:

```
backend=http://192.168.86.173:8001 input_chars=8000 approx_tokens=2000 total=500 success=500 errors=0 p99_ttfb=2.804165s p99_e2e=2.815436s
backend=http://192.168.86.176:8001 input_chars=8000 approx_tokens=2000 total=500 success=500 errors=0 p99_ttfb=2.743120s p99_e2e=2.754391s
```




## test concurrent with small tokens ->  max-num-seqs

```bash
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
)

SOURCE_URL="https://en.wikipedia.org/wiki/New_York_City"
TOTAL_REQUESTS=500
CONCURRENCY=80
INPUT_CHARS=300

INPUT_FILE=/tmp/vllm_embed_input.txt
PAYLOAD=/tmp/vllm_embed.json

curl -fsSL "$SOURCE_URL" | lynx -dump -stdin | iconv -f utf-8 -t utf-8 -c | head -c "$INPUT_CHARS" >"$INPUT_FILE"

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

  seq 1 "$TOTAL_REQUESTS" | xargs -P "$CONCURRENCY" -I{} bash -c '
    curl -sS -o /dev/null \
      -w "%{http_code} %{time_starttransfer} %{time_total}\n" \
      -X POST "$1/v1/embeddings" \
      -H "Content-Type: application/json" \
      --data-binary @"$2"
  ' _ "$ENDPOINT" "$PAYLOAD" >"$tmpfile"

  success=$(awk '$1 == "200" { c++ } END { print c + 0 }' "$tmpfile")
  total=$(wc -l <"$tmpfile" | tr -d " ")
  errors=$((total - success))

  p99_ttfb=$(awk '$1 == "200" { print $2 }' "$tmpfile" | percentile_99)
  p99_e2e=$(awk '$1 == "200" { print $3 }' "$tmpfile" | percentile_99)

  echo "backend=$ENDPOINT input_chars=$raw_chars approx_tokens=$approx_tokens total=$total success=$success errors=$errors p99_ttfb=${p99_ttfb}s p99_e2e=${p99_e2e}s"

  rm -f "$tmpfile"
done

rm -f "$INPUT_FILE" "$PAYLOAD"
```

```
backend=http://192.168.86.173:8001 input_chars=300 approx_tokens=75 total=500 success=500 errors=0 p99_ttfb=0.106968s p99_e2e=0.142153s
backend=http://192.168.86.176:8001 input_chars=300 approx_tokens=75 total=500 success=500 errors=0 p99_ttfb=0.151291s p99_e2e=0.231423s
```

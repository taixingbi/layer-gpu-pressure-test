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

## Load tests (prebuilt JSON — no per-request Python)

```bash
cat >/tmp/bench_embed.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail

############################################
# percentile
############################################
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

############################################
# config
############################################
BACKENDS=(
  "http://192.168.86.173:8001"
  "http://192.168.86.176:8001"
)

SIZES=(500 1000 1500 2000 2500 3000)

SOURCE_URL="https://en.wikipedia.org/wiki/New_York_City"
INPUT_FILE=/tmp/vllm_embed_input.txt
PAYLOAD=/tmp/vllm_embed.json

TOTAL=100
CONCURRENCY=20

############################################
# fetch source once
############################################
echo "Fetching source..."
RAW=/tmp/wiki_raw.txt
curl -fsSL "$SOURCE_URL" \
  | lynx -dump -stdin \
  | iconv -f utf-8 -t utf-8 -c > "$RAW"

############################################
# loop sizes
############################################
for SIZE in "${SIZES[@]}"; do
  echo ""
  echo "================ SIZE=${SIZE} ================="

  head -c "$SIZE" "$RAW" > "$INPUT_FILE"

  raw_chars=$(wc -c < "$INPUT_FILE" | tr -d ' ')
  tokens=$(( raw_chars / 4 ))

  echo "chars=$raw_chars approx_tokens=$tokens"

  ############################################
  # build payload
  ############################################
  python3 - <<PY
import json

with open("$INPUT_FILE", "r", encoding="utf-8", errors="ignore") as f:
    text = f.read()

payload = {"model": "BAAI/bge-m3", "input": text}

with open("$PAYLOAD", "w", encoding="utf-8") as out:
    json.dump(payload, out)
PY

  ############################################
  # test each backend
  ############################################
  for ENDPOINT in "${BACKENDS[@]}"; do
    tmpfile=$(mktemp)

    seq 1 $TOTAL | xargs -P $CONCURRENCY -I{} bash -c '
      curl -sS -o /dev/null \
        -w "%{time_starttransfer} %{time_total}\n" \
        -X POST "$1/v1/embeddings" \
        -H "Content-Type: application/json" \
        --data-binary @"$2"
    ' _ "$ENDPOINT" "$PAYLOAD" > "$tmpfile"

    p99_ttfb=$(awk '{print $1}' "$tmpfile" | percentile_99)
    p99_e2e=$(awk '{print $2}' "$tmpfile" | percentile_99)

    echo "backend=$ENDPOINT size=$SIZE tokens=$tokens p99_ttfb=${p99_ttfb}s p99_e2e=${p99_e2e}s"

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
backend=http://192.168.86.173:8001 p99_ttfb=0.112582s p99_e2e=0.124809s
backend=http://192.168.86.176:8001 p99_ttfb=0.379105s p99_e2e=0.651485s
```

## Large input load (8k chars, 500 reqs @ concurrency 80)

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

  echo "backend=$ENDPOINT total=$total success=$success errors=$errors p99_ttfb=${p99_ttfb}s p99_e2e=${p99_e2e}s"
  rm -f "$tmpfile"
done
```

Sample output:

```
backend=http://192.168.86.173:8001 total=500 success=500 errors=0 p99_ttfb=2.728527s p99_e2e=2.741782s
backend=http://192.168.86.176:8001 total=500 success=500 errors=0 p99_ttfb=2.670169s p99_e2e=2.683632s
```

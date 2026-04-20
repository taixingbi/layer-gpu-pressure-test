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
  "http://192.168.86.179:30181"
)

SOURCE_URL="https://en.wikipedia.org/wiki/New_York_City"
TOTAL_REQUESTS=200
CONCURRENCY_LEVELS=(20 60 100 140 180)

SMALL_INPUT_CHARS=2000
LARGE_INPUT_CHARS=8000

RAW_TEXT=/tmp/wiki_raw.txt
SMALL_TEXT=/tmp/wiki_small.txt
LARGE_TEXT=/tmp/wiki_large.txt
SMALL_PAYLOAD_FILE=/tmp/embed_small_payload.json
LARGE_PAYLOAD_FILE=/tmp/embed_large_payload.json

curl -fsSL "$SOURCE_URL" | lynx -dump -stdin | iconv -f utf-8 -t utf-8 -c >"$RAW_TEXT"

head -c "$SMALL_INPUT_CHARS" "$RAW_TEXT" >"$SMALL_TEXT"
head -c "$LARGE_INPUT_CHARS" "$RAW_TEXT" >"$LARGE_TEXT"

python3 - <<'PY'
import json

pairs = (
    ("/tmp/wiki_small.txt", "/tmp/embed_small_payload.json"),
    ("/tmp/wiki_large.txt", "/tmp/embed_large_payload.json"),
)

for text_path, payload_path in pairs:
    with open(text_path, "r", encoding="utf-8", errors="ignore") as f:
        payload = {"model": "BAAI/bge-m3", "input": f.read()}
    with open(payload_path, "w", encoding="utf-8") as out:
        json.dump(payload, out)
PY

for SIZE in small large; do
  if [ "$SIZE" = "small" ]; then
    PAYLOAD_FILE=$SMALL_PAYLOAD_FILE
  else
    PAYLOAD_FILE=$LARGE_PAYLOAD_FILE
  fi

  echo "=================================================="
  echo "size=$SIZE"
  echo "=================================================="

  for P in "${CONCURRENCY_LEVELS[@]}"; do
    echo "concurrency=$P"

    for ENDPOINT in "${BACKENDS[@]}"; do
      tmpfile=$(mktemp)

      seq 1 "$TOTAL_REQUESTS" | xargs -P "$P" -I{} bash -c '
        endpoint="$1"
        payload_file="$2"
        req_id="$3"

        curl -sS -o /dev/null \
          -w "%{http_code} %{time_starttransfer} %{time_total}\n" \
          -X POST "${endpoint}/v1/embeddings" \
          -H "X-Request-Id: request_id_${req_id}" \
          -H "X-Trace-Id: trace_id_${req_id}" \
          -H "X-Session-Id: session_id_${req_id}" \
          -H "Content-Type: application/json" \
          --data-binary @"${payload_file}"
      ' _ "$ENDPOINT" "$PAYLOAD_FILE" "{}" >"$tmpfile" 2>/dev/null

      success=$(awk '$1 == "200" { c++ } END { print c + 0 }' "$tmpfile")
      total=$(wc -l <"$tmpfile" | tr -d " ")
      errors=$((TOTAL_REQUESTS - success))

      p99_ttfb=$(awk '$1 == "200" { print $2 }' "$tmpfile" | percentile_99)
      p99_e2e=$(awk '$1 == "200" { print $3 }' "$tmpfile" | percentile_99)

      echo "$ENDPOINT size=$SIZE total=$TOTAL_REQUESTS success=$success errors=$errors p99_ttfb=${p99_ttfb}s p99_e2e=${p99_e2e}s"

      rm -f "$tmpfile"
    done

    echo
  done
done
```

Sample output (reference):

```
==================================================
backend=http://192.168.86.173:8001
==================================================
size=small concurrency=20 total=200 success=200 errors=0 p99_ttfb=0.376683s p99_e2e=0.394993s
size=small concurrency=60 total=200 success=200 errors=0 p99_ttfb=0.333300s p99_e2e=0.350056s
size=small concurrency=100 total=200 success=200 errors=0 p99_ttfb=0.490026s p99_e2e=0.508706s
size=small concurrency=140 total=200 success=200 errors=0 p99_ttfb=0.624375s p99_e2e=0.639639s
size=small concurrency=180 total=200 success=200 errors=0 p99_ttfb=0.612121s p99_e2e=0.661927s
size=large concurrency=20 total=200 success=200 errors=0 p99_ttfb=0.688027s p99_e2e=0.705028s
size=large concurrency=60 total=200 success=200 errors=0 p99_ttfb=2.039125s p99_e2e=2.053958s
size=large concurrency=100 total=200 success=200 errors=0 p99_ttfb=3.497601s p99_e2e=3.510997s
size=large concurrency=140 total=200 success=200 errors=0 p99_ttfb=4.867124s p99_e2e=4.878599s
size=large concurrency=180 total=200 success=200 errors=0 p99_ttfb=6.244714s p99_e2e=6.256547s
==================================================
backend=http://192.168.86.176:8001
==================================================
size=small concurrency=20 total=200 success=200 errors=0 p99_ttfb=0.321783s p99_e2e=0.339625s
size=small concurrency=60 total=200 success=200 errors=0 p99_ttfb=0.313291s p99_e2e=0.337063s
size=small concurrency=100 total=200 success=200 errors=0 p99_ttfb=0.495481s p99_e2e=0.515035s
size=small concurrency=140 total=200 success=200 errors=0 p99_ttfb=0.590922s p99_e2e=0.606990s
size=small concurrency=180 total=200 success=200 errors=0 p99_ttfb=0.613544s p99_e2e=0.628951s
size=large concurrency=20 total=200 success=200 errors=0 p99_ttfb=0.679921s p99_e2e=0.706791s
size=large concurrency=60 total=200 success=200 errors=0 p99_ttfb=1.998662s p99_e2e=2.011512s
size=large concurrency=100 total=200 success=200 errors=0 p99_ttfb=3.421086s p99_e2e=3.433411s
size=large concurrency=140 total=200 success=200 errors=0 p99_ttfb=4.754347s p99_e2e=4.767100s
size=large concurrency=180 total=200 success=200 errors=0 p99_ttfb=6.076810s p99_e2e=6.088401s
==================================================
backend=http://192.168.86.179:30181
==================================================
size=small concurrency=20 total=200 success=200 errors=0 p99_ttfb=0.201925s p99_e2e=0.342093s
size=small concurrency=60 total=200 success=200 errors=0 p99_ttfb=0.488511s p99_e2e=1.137211s
size=small concurrency=100 total=200 success=200 errors=0 p99_ttfb=1.044371s p99_e2e=1.507703s
size=small concurrency=140 total=200 success=200 errors=0 p99_ttfb=3.321560s p99_e2e=3.353747s
size=small concurrency=180 total=200 success=200 errors=0 p99_ttfb=1.113142s p99_e2e=1.540295s
size=large concurrency=20 total=200 success=200 errors=0 p99_ttfb=0.668183s p99_e2e=0.700451s
size=large concurrency=60 total=200 success=200 errors=0 p99_ttfb=2.006971s p99_e2e=2.032522s
size=large concurrency=100 total=200 success=200 errors=0 p99_ttfb=3.422610s p99_e2e=3.447401s
size=large concurrency=140 total=200 success=153 errors=47 p99_ttfb=4.143739s p99_e2e=4.172981s
size=large concurrency=180 total=200 success=137 errors=63 p99_ttfb=4.130601s p99_e2e=4.156007s
```

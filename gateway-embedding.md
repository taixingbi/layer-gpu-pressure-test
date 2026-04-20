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

INPUT_CHARS=2000

RAW_TEXT=/tmp/wiki_raw.txt
TEXT=/tmp/wiki_small.txt
PAYLOAD_FILE=/tmp/embed_small_payload.json

# prepare text
curl -fsSL "$SOURCE_URL" | lynx -dump -stdin | iconv -f utf-8 -t utf-8 -c >"$RAW_TEXT"
head -c "$INPUT_CHARS" "$RAW_TEXT" >"$TEXT"

# build payload
python3 - <<'PY'
import json
with open("/tmp/wiki_small.txt", "r", encoding="utf-8", errors="ignore") as f:
    payload = {"model": "BAAI/bge-m3", "input": f.read()}
with open("/tmp/embed_small_payload.json", "w", encoding="utf-8") as out:
    json.dump(payload, out)
PY

echo "================ SMALL TEST ================"

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





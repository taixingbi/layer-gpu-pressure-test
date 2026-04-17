# layer-gpu-pressure-test

## reference
### k3s
```bash
for p in 60 80 100 120; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:30080/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "{\"model\":\"Qwen/Qwen2.5-7B-Instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"introduce new york city\"}],
\"max_tokens\":256}"
  ' | sort | uniq -c
done
```

### gateway-inference

```bash
# dev
for p in 20 40 60; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:30180/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "{\"model\":\"Qwen/Qwen2.5-7B-Instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"introduce new york city\"}],
\"max_tokens\":256}"
  ' | sort | uniq -c
done

# prod small reqeust
for p in 20 40 60 80 100 120; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:30380/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "{\"model\":\"Qwen/Qwen2.5-7B-Instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"introduce new york city\"}],
\"max_tokens\":56}"
  ' | sort | uniq -c
done


# prod large reqeust
for p in 20 40 60; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:30380/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "{\"model\":\"Qwen/Qwen2.5-7B-Instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"introduce new york city\"}],
\"max_tokens\":256}"
  ' | sort | uniq -c
done

# prod mixed reqeust
SMALL='{"model":"Qwen/Qwen2.5-7B-Instruct","messages":[{"role":"user","content":"introduce new york city"}],"max_tokens":64}'
LARGE='{"model":"Qwen/Qwen2.5-7B-Instruct","messages":[{"role":"user","content":"Write a detailed 8-section travel guide for New York City including history, neighborhoods, transportation, food, attractions, itinerary, budget tips, and safety advice."}],"max_tokens":512}'

for p in 40 60 80; do
  echo "=== mixed random concurrency $p ==="

  seq 1 200 | xargs -I{} -P $p bash -c '
    r=$((RANDOM % 4))
    if [ $r -eq 0 ]; then
      TYPE="large"
      DATA='"'"$LARGE"'"'
    else
      TYPE="small"
      DATA='"'"$SMALL"'"'
    fi

    code=$(curl -s -o /dev/null -w "%{http_code}" http://192.168.86.179:30380/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "$DATA")

    echo "$TYPE $code"
  ' | sort | uniq -c

done
```


## gate-embed
```bash
# small 
for p in 20 60 100 140 180; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://localhost:30181/v1/embeddings \
      -H "Content-Type: application/json" \
      -H "X-Trace-Id: trace_id_1"
      -H "X-Request-Id: request_id_1" \
      -H "X-Session-Id: session_id_1" \
      -d "{\"model\":\"BAAI/bge-m3\",\"input\":\"New York City comprises 5 boroughs sitting where the Hudson River meets the Atlantic Ocean. At its core is Manhattan, a densely populated borough that’s among the world’s major commercial, financial and cultural centers. Its iconic sites include skyscrapers such as the Empire State Building and sprawling Central Park. Broadway theater is staged in neon-lit Times Square\"}"
  ' | sort | uniq -c
done
```

```bash
# large 
URL="http://192.168.86.179:30181/v1/embeddings"
URL="http://localhost:30181/v1/embeddings"

HEADERS=(
  -H "Content-Type: application/json"
  -H "X-Trace-Id: trace_id_1"
  -H "X-Request-Id: request_id_1"
  -H "X-Session-Id: session_id_1"
)

PAYLOAD=$(cat <<'EOF'
{
  "model": "BAAI/bge-m3",
  "input": "New York City is one of the most dynamic and influential urban centers in the world, known for its diversity, cultural significance, and economic power. The city comprises five boroughs: Manhattan, Brooklyn, Queens, the Bronx, and Staten Island. Each borough has its own unique identity, history, and role within the broader metropolitan area. Manhattan is often considered the heart of the city, home to iconic landmarks such as Times Square, Central Park, Wall Street, and the Empire State Building. It is a global hub for finance, media, entertainment, and technology. Brooklyn has evolved into a cultural hotspot known for its artistic communities, diverse neighborhoods, and vibrant food scene. Queens is the largest borough by area and one of the most ethnically diverse places on Earth. The Bronx is known for its rich history and contributions to music and sports, including being the birthplace of hip-hop. Staten Island offers a quieter suburban atmosphere and scenic views of the Statue of Liberty. New York City's economy is one of the largest in the world, driven by finance, technology, healthcare, education, and tourism. Its public transportation system enables millions of people to move efficiently every day. Culturally, the city is unmatched, with Broadway theaters, world-class museums, and countless music venues. Education is supported by institutions such as Columbia University and New York University. Tourism attracts millions each year to landmarks like the Statue of Liberty, Central Park, and the Brooklyn Bridge. The food scene reflects global diversity, from street vendors to Michelin-starred restaurants. The city’s history dates back to its founding as New Amsterdam. Today, it continues to evolve, balancing historical roots with modern development. Challenges include housing affordability, transportation infrastructure, and climate change. Despite these, New York City remains a symbol of opportunity, resilience, and ambition, influencing global trends in media, fashion, and finance."
}
EOF
)

for p in 20 60 100 140 180; do
  echo "=== concurrency $p ==="

  seq 200 | xargs -P "$p" -I{} \
    curl -s -o /dev/null -w "%{http_code}\n" \
    "$URL" "${HEADERS[@]}" \
    -d "$PAYLOAD" \
  | sort | uniq -c

done
```

```bash
# mixed large 
URL="http://192.168.86.179:30181/v1/embeddings"
URL="http://localhost:30181/v1/embeddings"

HEADERS=(
  -H "Content-Type: application/json"
  -H "X-Trace-Id: trace_id_1"
  -H "X-Request-Id: request_id_1"
  -H "X-Session-Id: session_id_1"
)

SMALL_PAYLOAD='{"model":"BAAI/bge-m3","input":"New York City comprises 5 boroughs sitting where the Hudson River meets the Atlantic Ocean. At its core is Manhattan, a densely populated borough that is among the world'\''s major commercial, financial, and cultural centers. Its iconic sites include skyscrapers such as the Empire State Building and sprawling Central Park. Broadway theater is staged in neon-lit Times Square."}'

LARGE_TEXT=$(python3 - <<'PY'
text = """
New York City is one of the most dynamic and influential urban centers in the world, known for its diversity, cultural significance, and economic power. The city comprises five boroughs: Manhattan, Brooklyn, Queens, the Bronx, and Staten Island. Each borough has its own unique identity, history, and role within the broader metropolitan area.
"""
words = text.split()
target = 5000
result = (words * (target // len(words) + 1))[:target]
print(" ".join(result))
PY
)

LARGE_PAYLOAD=$(jq -n --arg input "$LARGE_TEXT" '{model:"BAAI/bge-m3", input:$input}')

for p in 20 60 100 140 180; do
  echo "=== SMALL concurrency $p ==="
  seq 200 | xargs -P "$p" -I{} \
    curl -s -o /dev/null -w "%{http_code}\n" \
    "$URL" "${HEADERS[@]}" \
    -d "$SMALL_PAYLOAD" \
  | sort | uniq -c

  echo "=== LARGE concurrency $p ==="
  seq 200 | xargs -P "$p" -I{} \
    curl -s -o /dev/null -w "%{http_code}\n" \
    "$URL" "${HEADERS[@]}" \
    -d "$LARGE_PAYLOAD" \
  | sort | uniq -c
done
```



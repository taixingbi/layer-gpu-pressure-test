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
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

### gateway

```bash
for p in 20 40 60 80 100 120; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:8010/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "{\"model\":\"Qwen/Qwen2.5-7B-Instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"introduce new york city\"}],
\"max_tokens\":256}"
  ' | sort | uniq -c
done
```


## embed
```bash
for p in 20 40 60 80 100 120; do
  echo "=== concurrency $p ==="
  seq 1 200 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:8011/v1/embeddings \
      -H "Content-Type: application/json" \
      -H "X-Internal-Key: 1234" \
      -H "X-Request-Id: request_id_1" \
      -H "X-Session-Id: session_id_1" \
      -d "{\"model\":\"BAAI/bge-m3\",\"input\":\"New York City comprises 5 boroughs sitting where the Hudson River meets the Atlantic Ocean. At its core is Manhattan, a densely populated borough that’s among the world’s major commercial, financial and cultural centers. Its iconic sites include skyscrapers such as the Empire State Building and sprawling Central Park. Broadway theater is staged in neon-lit Times Square\"}"
  ' | sort | uniq -c
done
```

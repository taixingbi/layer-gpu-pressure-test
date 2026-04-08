# layer-gpu-pressure-test

### k3s
```bash
for p in 10 20 30 40 50; do
  echo "=== concurrency $p ==="
  seq 1 100 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:30080/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "{\"model\":\"Qwen/Qwen2.5-7B-Instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"where is jersey city\"}],\"max_tokens\":50}"
  ' | sort | uniq -c
done
```

### gateway

```bash
for p in 10 20 30 40 50; do
  echo "=== concurrency $p ==="
  seq 1 100 | xargs -I{} -P $p sh -c '
    curl -s -o /dev/null -w "%{http_code}\n" http://192.168.86.179:8010/v1/chat/completions \
      -H "Content-Type: application/json" \
      -d "{\"model\":\"Qwen/Qwen2.5-7B-Instruct\",\"messages\":[{\"role\":\"user\",\"content\":\"where is jersey city\"}],\"max_tokens\":50}"
  ' | sort | uniq -c
done
```

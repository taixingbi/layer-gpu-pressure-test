# layer-gpu-pressure-test

```bash
seq 1 100 | xargs -I{} -P 50 curl -s http://192.168.86.179:8010/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "Qwen/Qwen2.5-7B-Instruct", "messages": [{"role": "user", "content": "where is jersey city"}], "max_tokens": 50}' \
  > /dev/null
```

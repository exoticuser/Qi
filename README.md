# Qi — Self-hosted LLM via GitHub Actions

This repository provides a GitHub Actions workflow that downloads the
[`mradermacher/Llama3.3-8B-Instruct-Thinking-Heretic-Uncensored-Claude-4.5-Opus-High-Reasoning-i1-GGUF`](https://huggingface.co/mradermacher/Llama3.3-8B-Instruct-Thinking-Heretic-Uncensored-Claude-4.5-Opus-High-Reasoning-i1-GGUF)
model from Hugging Face and serves it with an **OpenAI-compatible REST API**
accessible over a public cloudflared tunnel — all running entirely inside a
GitHub Actions runner.

---

## How to run

1. Go to **Actions → Serve LLM Model** in this repository.
2. Click **Run workflow**.
3. Choose a quantization level and how long to keep the server alive.
4. Once the **"Expose endpoint"** step finishes, scroll through its log output.
   You will see a block like:

```
============================================================
  🚀  LLM API Server is LIVE
============================================================

  Base URL : https://xxxx-xxxx-xxxx.trycloudflare.com

  Endpoints
  ---------
  GET  https://xxxx-xxxx-xxxx.trycloudflare.com/health
  GET  https://xxxx-xxxx-xxxx.trycloudflare.com/v1/models
  POST https://xxxx-xxxx-xxxx.trycloudflare.com/v1/chat/completions
  POST https://xxxx-xxxx-xxxx.trycloudflare.com/v1/completions
…
============================================================
```

Copy the base URL from that log and use it with the examples below.

---

## API endpoints

All endpoints follow the [OpenAI API](https://platform.openai.com/docs/api-reference) format.
Replace `BASE_URL` with the `https://xxxx.trycloudflare.com` URL from the workflow log.

### Health check

```bash
GET BASE_URL/health
```

```bash
curl BASE_URL/health
# → {"status":"ok"}
```

---

### List models

```bash
GET BASE_URL/v1/models
```

```bash
curl BASE_URL/v1/models
```

Sample response:

```json
{
  "object": "list",
  "data": [
    {
      "id": "llm",
      "object": "model",
      "created": 0,
      "owned_by": "llamacpp"
    }
  ]
}
```

---

### Chat completions  *(recommended)*

```bash
POST BASE_URL/v1/chat/completions
Content-Type: application/json
```

```bash
curl BASE_URL/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llm",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user",   "content": "Explain quantum entanglement simply."}
    ],
    "max_tokens": 512,
    "temperature": 0.7
  }'
```

Sample response:

```json
{
  "id": "chatcmpl-...",
  "object": "chat.completion",
  "created": 1234567890,
  "model": "llm",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "Quantum entanglement is …"
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 32,
    "completion_tokens": 150,
    "total_tokens": 182
  }
}
```

---

### Text completions

```bash
POST BASE_URL/v1/completions
Content-Type: application/json
```

```bash
curl BASE_URL/v1/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llm",
    "prompt": "Once upon a time in a land far away",
    "max_tokens": 256,
    "temperature": 0.8
  }'
```

---

### Streaming responses

Both `/v1/chat/completions` and `/v1/completions` support streaming via
`"stream": true`:

```bash
curl BASE_URL/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llm",
    "messages": [{"role": "user", "content": "Count to 10."}],
    "stream": true
  }'
```

---

## Workflow inputs

| Input | Default | Description |
|---|---|---|
| `quantization` | `Q4_K_M` | GGUF quantization level. `Q2_K` ≈ 2.9 GB RAM, `Q3_K_M` ≈ 3.9 GB RAM, `Q4_K_M` ≈ 4.8 GB RAM, `Q5_K_M` ≈ 5.7 GB RAM. |
| `keep_alive_minutes` | `60` | How long to keep the server running (1–350 minutes). |

## Secrets (optional)

| Secret | Description |
|---|---|
| `HF_TOKEN` | Hugging Face token. Only required if the model is gated/private. |

To add it: **Settings → Secrets and variables → Actions → New repository secret**.

---

## Using with OpenAI-compatible clients

Because the API is fully OpenAI-compatible you can point any OpenAI SDK at it:

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://xxxx-xxxx-xxxx.trycloudflare.com/v1",
    api_key="not-required",   # llama-server doesn't enforce API keys
)

response = client.chat.completions.create(
    model="llm",
    messages=[{"role": "user", "content": "What is the meaning of life?"}],
    max_tokens=256,
)
print(response.choices[0].message.content)
```

---

## Notes

- The server runs on the **GitHub Actions runner** (2 vCPUs, 7 GB RAM, Ubuntu).
- The **cloudflared quick tunnel** URL is temporary and changes every run.
- The server shuts down when the workflow finishes or the keep-alive timer expires.
- For long-running or production use, consider deploying on dedicated hardware.
# SemSummarize Service

## LLM-powered entity summarization

---

## What is SemSummarize?

AI-powered summarization service using any OpenAI API-compatible LLM (local or cloud).

**Examples:** Ollama, LM Studio, OpenAI API, Anthropic, etc.

---

## Setup Examples

### Ollama (Local)

```bash
docker run -d -p 11434:11434 ollama/ollama
ollama pull llama2
```

### LM Studio (Local GUI)

1. Download from https://lmstudio.ai
2. Load a model (e.g., Mistral 7B)
3. Start server (default port: 1234)

### OpenAI API (Cloud)

Set API key in environment:

```bash
export OPENAI_API_KEY=sk-...
```

---

## Configuration

```json
{
  "graph": {
    "config": {
      "summarization": {
        "enabled": true,
        "service_url": "http://localhost:11434/v1",
        "model": "llama2",
        "api_key": "${OPENAI_API_KEY}",
        "max_tokens": 150,
        "temperature": 0.7
      }
    }
  }
}
```

---

## Usage

### Automatic

When enabled, graph processor generates summaries for entities:

```json
{
  "id": "UAV-002",
  "properties": {"battery": 15.4, "status": "critical"},
  "summary": "UAV-002 is in critical condition with battery at 15.4%, requiring immediate attention."
}
```

### Manual API

```bash
curl -X POST http://localhost:8080/graph/entity/UAV-002/summarize
```

---

## Next Steps

- **[Decision Guide](04-decision-guide.md)** - When to use summarization
- **[SemEmbed](02-semembed.md)** - Enhanced embeddings

---

**SemSummarize enriches entities** - Use for human-readable summaries of complex entity state.

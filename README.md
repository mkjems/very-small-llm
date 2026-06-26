# very-small-llm

This project provides a very small LLM service for my other web projects.

The idea is to run the LLM in a container on the VPS and expose it only on
localhost, so other local apps can call it through an API.

For deployment context and collaboration notes, see the docs in `docs/`.

## Local experiments

Ollama:

```sh
docker compose up -d
docker exec very-small-llm-ollama ollama pull qwen3:0.6b
curl http://127.0.0.1:11434/api/tags
```

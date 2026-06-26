# TODO

This file is the active plan for moving `very-small-llm` forward. Completed
plans can be moved to `docs/completed-plans/` later, but this file should stay
focused on the next useful steps.

## Guiding Goal

Run a very small local LLM on the VPS that other projects on the same VPS can
call through an internal API. Start with the easiest working proof of concept,
then only make it leaner or more production-like after we can measure the tradeoffs.

## Questions We Need To Answer

- [x] What VPS resources do we actually have available: CPU cores, RAM, disk, and
      whether there is any GPU?
      Answer: 2 vCPU, 4 GB RAM, 40 GB Disk local, No GPU. No other volumes at the moment, but can be bought cheaply 
- [x] What response quality is "good enough" for the first real use case?
      Answer:  This is more of an experiment. Lets find out what quality we get.   
- [x] What latency is acceptable for the first real use case?
      Answer: Lets find out what we get.  
- [ ] Should downstream apps call the model runtime directly, or should we add a
      small stable wrapper API in front of it?
      Answer: I don't know what you mean or want to achieve with that.
- [x] Do we need the model to be private to the VPS only, or should there ever be
      public access through Caddy?
      Answer: it is private.

## Current Constraints

- VPS: 2 vCPU, 4 GB RAM, 40 GB local disk, no GPU.
- Runtime should be CPU-first.
- Service should stay private on localhost.
- Start with tiny models only.
- Disk is limited but enough for the Ollama runtime plus several tiny models.
- Avoid deploying the full Ollama image to the VPS unless llama.cpp proves too
  awkward. The Ollama image is convenient but large for this server.

## Human Learning Track

The human should not need to become an ML engineer, but should understand enough
to make good deployment decisions.

- [ ] Learn the basic vocabulary: model weights, quantization, tokens, context
      length, prompt, system prompt, temperature, streaming, embeddings.
- [ ] Learn the operational tradeoff: smaller models are cheaper and faster, but
      less capable; bigger models need more RAM/VRAM and may be too slow on CPU.
- [ ] Learn why a volume matters: model files are large and should survive
      container restarts, image updates, and VPS reboots.
- [ ] Learn the difference between Ollama and llama.cpp:
      Ollama is easier to operate and manages models for us; llama.cpp is closer
      to the metal and can be leaner once we know exactly what we need.
- [ ] Learn how to judge a model for this project: response quality, tokens per
      second, memory use, cold-start time, disk use, and API compatibility.

## Current Direction

Since `ghcr.io/ggml-org/llama.cpp:server` is already running locally with
`Qwen/Qwen3-0.6B-GGUF:Q8_0`, make llama.cpp the primary VPS candidate and treat
Ollama as optional comparison material.

Local observation:
the running local llama.cpp server is reachable on `127.0.0.1:8080` with model
alias `qwen`. A first chat completion smoke test worked and returned
approximately 95 generated tokens/second for a 37-token response.

Ollama observation:
Ollama is also running locally on `127.0.0.1:11434`, but `/api/tags` currently
now has `smollm2:135m` installed. The model is about 270 MB, with 134.52M
parameters, F16 quantization, and 8192 context length. A first smoke test worked,
but the answer quality looked weak/odd for a plain introduction prompt.

## T1 - Optional POC Using `ollama/ollama:latest`

Ollama is useful because it minimizes yak shaving, but the image is large for
the VPS. Keep this track as optional comparison material unless llama.cpp gets
stuck.

### T1.1 - Run `ollama/ollama:latest` On Local Machine

- [x] Confirm `compose.yml` starts the Ollama container bound to
      `127.0.0.1:11434`.
- [x] Pull the smallest sanity-check model first: `smollm2:135m`.
      Observed model size: about 270 MB.
- [ ] Pull a more realistic tiny baseline after that: `qwen3:0.6b`.
- [ ] Compare both before deciding which one is worth trying on the VPS.
- [x] Add a smoke-test command for listing local models.
      Command: `curl http://127.0.0.1:11434/api/tags`
- [x] Add a smoke-test command for one non-streaming generation request.
      Command: `curl http://127.0.0.1:11434/api/generate`
- [ ] Record local observations in this file or a small notes doc:
      model size, startup time, first response latency, tokens per second, and
      whether the answer quality feels usable.
- [ ] Decide whether local testing is good enough to try the same runtime on the
      VPS.

### T1.2 - Run `ollama/ollama:latest` On VPS

- [ ] Create or choose the VPS app directory, likely `/opt/very-small-llm`.
- [ ] Copy or deploy the Compose file to the VPS.
- [ ] Use a persistent Docker volume for `/root/.ollama`.
- [ ] Bind the service to `127.0.0.1:11434`, not `0.0.0.0`.
- [ ] Pull the chosen baseline model on the VPS.
- [ ] Smoke test from the VPS host with `curl http://127.0.0.1:11434/api/tags`.
- [ ] Smoke test a non-streaming generation request from the VPS host.
- [ ] Check memory, CPU, disk use, and response speed while generating.
- [ ] Document VPS-specific commands and observations.
- [ ] Decide whether the VPS can comfortably run this model for real projects.

### T1.3 - Define The First Internal API Contract

The first version can use Ollama's API directly, but we should decide whether
that is a temporary shortcut or the long-term contract for other apps.

- [ ] List the first consuming project or projects.
- [ ] Define the minimum request shape they need: prompt only, chat messages,
      system prompt, JSON output, streaming, or not streaming.
- [ ] Define the minimum response shape they should rely on.
- [ ] Decide on timeouts and max generated tokens for callers.
- [ ] Decide whether callers should know the model name, or whether this service
      should hide that behind a stable default.
- [ ] If needed, plan a tiny wrapper service that exposes our own stable API and
      forwards to Ollama internally.

Wrapper API explanation:
without a wrapper, other apps call Ollama directly and become coupled to
Ollama's request/response format and model names. With a wrapper, other apps call
our small stable endpoint, and we can later switch from Ollama to llama.cpp
without changing every consuming app. For the first experiment, direct Ollama
calls are fine.

### T1.4 - Build A Tiny Evaluation Set

Before changing runtimes or models, we need a repeatable way to compare them.

- [ ] Write 5-10 prompts that represent real project use cases.
- [ ] Include at least one prompt that asks for structured JSON if that matters
      for downstream apps.
- [ ] Save expected qualities for each prompt, not just exact answers.
- [ ] Run the same prompts against each candidate model/runtime.
- [ ] Record latency, tokens per second, and rough answer quality.

## T2 - Make The Leaner Version Using Raw `llama.cpp`

This is now the primary path. The purpose is to get a smaller, faster, simpler
production runtime than Ollama.

### T2.1 - Run `llama.cpp` Server Locally

- [x] Start `ghcr.io/ggml-org/llama.cpp:server` locally with
      `Qwen/Qwen3-0.6B-GGUF:Q8_0`.
- [x] Confirm a local llama.cpp server is reachable at `127.0.0.1:8080`.
- [x] Decide whether this repo's `compose-llamacpp.yml` should use host port
      `8080` or keep `8083` to avoid colliding with other local apps.
      Decision: keep repo-managed llama.cpp on host port `8083`.
- [ ] Confirm the selected GGUF model downloads or mounts correctly.
- [x] Smoke test the OpenAI-compatible chat completion endpoint.
- [ ] Run the same tiny evaluation set from T1.4.
- [ ] Compare memory use, disk use, startup time, latency, and answer quality
      against Ollama.

### T2.2 - Decide Between Ollama And `llama.cpp`

- [ ] Keep Ollama if ease of operations matters more than shaving runtime weight.
- [ ] Move to `llama.cpp` if it is meaningfully lighter, fast enough, and the API
      compatibility is good enough for our callers.
- [ ] Document the decision and the reason, so we do not re-litigate it later
      without new evidence.

## T3 - Production Shape On The VPS

This stage turns the chosen runtime into a service that is boring to operate.

- [ ] Create a VPS deployment doc for this project.
- [ ] Decide whether deployment belongs in this repo, the infrastructure repo, or
      both.
- [ ] Add restart policy, health checks, and resource limits where practical.
- [ ] Document model storage location, volume name, and backup expectations.
- [ ] Document how to update the container image and model.
- [ ] Document rollback steps.
- [ ] Add basic log inspection commands.
- [ ] Keep the service private to localhost unless we explicitly decide otherwise.

## T4 - Integration With Other VPS Projects

- [ ] Pick one real consuming app for the first integration.
- [ ] Add environment variables in that app for the local LLM base URL, timeout,
      and model/default behavior.
- [ ] Add one narrow feature that calls the LLM.
- [ ] Handle failures gracefully: timeout, model unavailable, invalid JSON, or
      low-quality output.
- [ ] Decide whether shared client code or examples belong in this repo.

## T5 - Future Improvements

Do not start these until the basic local/VPS loop works.

- [ ] Add a wrapper API if direct runtime APIs become awkward.
- [ ] Add request logging that avoids storing sensitive prompts by default.
- [ ] Add simple rate limiting or concurrency limits if callers can overload the
      VPS.
- [ ] Explore embeddings only if a real use case appears.
- [ ] Explore larger models only after we know the VPS can handle the baseline.
- [ ] Consider a public endpoint only with authentication, explicit Caddy routing,
      and a clear reason.

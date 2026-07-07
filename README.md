# modelship-unraid

Unraid Community Applications (CA) template for [modelship](https://github.com/alez007/modelship) — a self-hosted, OpenAI-compatible inference server (chat/reasoning, embeddings, STT, TTS, image generation) built on Ray Serve, with pluggable vLLM / llama.cpp / Diffusers backends.

This repo just holds the CA template XML — the application itself lives in the main [modelship](https://github.com/alez007/modelship) repo.

## Installing

### Option A — add as a Template Repository (recommended)

1. Unraid → **Apps** (Community Applications) → **Settings** → **Template Repositories**.
2. Add: `https://github.com/alez007/modelship-unraid`
3. Save. The `modelship` template now shows up under Apps → search "modelship", or directly under Docker → **Add Container** → Template dropdown.

### Option B — one-off manual install

Docker tab → **Add Container** → Template field → paste:
`https://raw.githubusercontent.com/alez007/modelship-unraid/main/templates/modelship.xml`

## GPU vs CPU-only

The template defaults to the **GPU image** (`ghcr.io/alez007/modelship:latest`), which needs the NVIDIA Driver plugin and NVIDIA Container Toolkit set up on your Unraid box.

For a **CPU-only** install (works on any box, no GPU needed):

1. After adding the container (before or after first start), edit it.
2. Change **Repository** from `ghcr.io/alez007/modelship:latest` to `ghcr.io/alez007/modelship:latest-cpu`.
3. In **Extra Parameters**, remove `--runtime=nvidia`.
4. Remove (or ignore) the `NVIDIA_VISIBLE_DEVICES` / `NVIDIA_DRIVER_CAPABILITIES` variables — they're no-ops without the NVIDIA runtime.

## Before you start the container

modelship refuses to start without a `models.yaml`. Create one at the path you set for **Models Config** (default `/mnt/user/appdata/modelship/models.yaml`) before applying the template.

Smallest possible CPU quick-start (a tiny reasoning model, no GPU, runs almost anywhere):

```yaml
models:
  - name: reasoning-qwen
    model: "lmstudio-community/Qwen3-0.6B-GGUF:*Q4_K_M.gguf"
    usecase: generate
    loader: llama_server
    num_cpus: 3
    llama_server_config:
      n_ctx: 4096
```

For GPU models (vLLM, Diffusers), multi-model stacks, and the full config reference, see [docs/model-configuration.md](https://github.com/alez007/modelship/blob/main/docs/model-configuration.md) and the ready-made examples in [config/examples/](https://github.com/alez007/modelship/tree/main/config/examples).

## `--shm-size` — read this before raising it

The template's `--shm-size=2g` default (in **Extra Parameters**) is only sized for the single small quick-start model above. Ray sizes its shared-memory object store to roughly **30% of the RAM it detects as available, at startup** — before any model even loads. If `--shm-size` is smaller than that computed target, Ray crashes immediately on boot.

There's no universal "correct" number — it scales with your box's RAM and what you're running:

- Bigger/more models, or a GPU stack → raise `--shm-size` to roughly 30% of the RAM you want to give modelship.
- Small Unraid boxes with little free RAM → keep it low; setting it far above what the host actually has doesn't crash anything immediately (the tmpfs is lazily backed), but if Ray/vLLM actually try to use that much, you'll hit real out-of-memory pressure instead of a clean startup error.

## What's exposed vs not

This template keeps the Config surface intentionally small — enough to get a working single-container deployment without needing to understand Ray, state stores, or gateway replicas:

- **Always visible**: API port, models.yaml path, model cache path, `HF_TOKEN`.
- **Advanced**: metrics port + toggle, `MSHIP_API_KEYS`, `MSHIP_LOG_LEVEL`, and (GPU variant) `NVIDIA_VISIBLE_DEVICES`/`NVIDIA_DRIVER_CAPABILITIES`.
- **Not exposed** (production/multi-node knobs that don't matter for a single Unraid container — defaults are fine, and you can still add any of these manually as extra Variables if you need them): gateway name/replicas/concurrency, state store (Redis/file), OpenTelemetry export, syslog log target/format, existing-Ray-cluster attach, Ray session pruning, preflight toggle, request body size limit, Ray head CPU/GPU pinning, Ray dashboard. Plugin backends (Kokoro ONNX, Orpheus, whisper.cpp) need no container-level config at all — they're pulled in automatically based on what your `models.yaml` references.

## Icon

`templates/modelship.xml` references [`icon.svg`](icon.svg) in this repo.

## License

Apache 2.0, matching [modelship](https://github.com/alez007/modelship).

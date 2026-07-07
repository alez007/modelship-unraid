# modelship-unraid

Unraid Docker template for [modelship](https://github.com/alez007/modelship) — a self-hosted, OpenAI-compatible inference server (chat/reasoning, embeddings, STT, TTS, image generation) built on Ray Serve, with pluggable vLLM / llama.cpp / Diffusers backends.

This repo just holds the template XML — the application itself lives in the main [modelship](https://github.com/alez007/modelship) repo.

## Installing

Unraid removed the OS-level "Template Repositories" setting in 6.10.0 (per Squid, the Community Applications plugin developer: *"The Template Repositories section of the OS is now removed in 6.10.0+. It is not coming back."*). There is no field anywhere where you paste this repo's URL and its template just shows up — that mechanism doesn't exist anymore. Instead, install the template file itself, manually, using one of these:

Grab the raw XML: `https://raw.githubusercontent.com/alez007/modelship-unraid/main/templates/modelship.xml`

**Option A — plain Docker (no plugin needed):**

1. Save it as `modelship.xml` in `/boot/config/plugins/dockerMan/templates-user/` on your Unraid box (Unraid's built-in File Manager, or `wget` over SSH into that folder).
2. Docker tab → **Add Container** → **Template** dropdown → `modelship` is now listed.

**Option B — Community Applications "Private" templates (if you have the CA plugin):**

Save the same XML instead to `/boot/config/plugins/community.applications/private/<your-username>/modelship.xml`. It'll show up under Apps → search, tagged "Private" — same manual copy, just a bit more integrated with the CA browsing UI.

Either way this is local to your box only — it does **not** make the template discoverable by other Unraid users. That requires actually submitting it to the official CA index for review (a forum post or PR to [Squidly271/community.applications](https://github.com/Squidly271/community.applications)), which is a separate, out-of-scope step from this repo.

## GPU vs CPU-only

This is a single template (`modelship`) that offers **two image tags** via Unraid's tag/branch selector — a dropdown CA shows once you pick the template, before you fill in the rest of the form:

- **`latest`** (default) — GPU build, needs the NVIDIA Driver plugin and NVIDIA Container Toolkit set up on your Unraid box.
- **`latest-cpu`** — CPU-only build, works on any box (amd64 or arm64), no GPU needed.

**Important limitation:** tag selection only swaps the image tag — it can't conditionally change Extra Parameters or hide/show variables. If you pick `latest-cpu`, you must also manually remove `--runtime=nvidia` from **Extra Parameters** (toggle **Advanced View** in Add Container) before creating the container, or it'll fail to start (Docker errors out immediately if `--runtime=nvidia` is set but the NVIDIA runtime isn't registered). The `NVIDIA_VISIBLE_DEVICES`/`NVIDIA_DRIVER_CAPABILITIES` variables are harmless no-ops on the CPU tag and can be left alone or removed — only `--runtime=nvidia` actually breaks things.

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
- **Advanced**: metrics port + toggle, `MSHIP_API_KEYS`, `MSHIP_LOG_LEVEL`, and `NVIDIA_VISIBLE_DEVICES`/`NVIDIA_DRIVER_CAPABILITIES` (relevant on the `latest` GPU tag; no-ops on `latest-cpu`).
- **Not exposed** (production/multi-node knobs that don't matter for a single Unraid container — defaults are fine, and you can still add any of these manually as extra Variables if you need them): gateway name/replicas/concurrency, state store (Redis/file), OpenTelemetry export, syslog log target/format, existing-Ray-cluster attach, Ray session pruning, preflight toggle, request body size limit, Ray head CPU/GPU pinning, Ray dashboard. Plugin backends (Kokoro ONNX, Orpheus, whisper.cpp) need no container-level config at all — they're pulled in automatically based on what your `models.yaml` references.

## Icon

`templates/modelship.xml` references [`icon.svg`](icon.svg) in this repo.

## License

Apache 2.0, matching [modelship](https://github.com/alez007/modelship).

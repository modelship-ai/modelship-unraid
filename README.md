# modelship-unraid

Unraid Docker template for [modelship](https://github.com/modelship-ai/modelship) — a self-hosted, OpenAI-compatible inference server (chat/reasoning, embeddings, STT, TTS, image generation) built on Ray Serve, with pluggable vLLM / llama.cpp / Diffusers backends.

This repo just holds the template XML — the application itself lives in the main [modelship](https://github.com/modelship-ai/modelship) repo.

## Installing

Unraid removed the OS-level "Template Repositories" setting in 6.10.0 (per Squid, the Community Applications plugin developer: *"The Template Repositories section of the OS is now removed in 6.10.0+. It is not coming back."*). There is no field anywhere where you paste this repo's URL and its template just shows up — that mechanism doesn't exist anymore. Instead, install the template file itself, manually, using one of these:

There are two separate templates — grab whichever variant(s) you want (see [GPU vs CPU-only](#gpu-vs-cpu-only) below):

- GPU: `https://raw.githubusercontent.com/modelship-ai/modelship-unraid/main/templates/modelship-cuda.xml`
- CPU-only: `https://raw.githubusercontent.com/modelship-ai/modelship-unraid/main/templates/modelship-cpu.xml`

**Option A — plain Docker (no plugin needed):**

1. Save the XML file(s) into `/boot/config/plugins/dockerMan/templates-user/` on your Unraid box (Unraid's built-in File Manager, or `wget` over SSH into that folder) — keep the filenames as-is.
2. Docker tab → **Add Container** → **Template** dropdown → `modelship-cuda` and/or `modelship-cpu` are now listed.

**Option B — Community Applications "Private" templates (if you have the CA plugin):**

Save the same XML file(s) instead to `/boot/config/plugins/community.applications/private/<your-username>/`. They'll show up under Apps → search, tagged "Private" — same manual copy, just a bit more integrated with the CA browsing UI.

Either way this is local to your box only — it does **not** make the template discoverable by other Unraid users. That requires submitting this repo through the official [Community Apps portal](https://ca.unraid.net/submit), which is a separate, out-of-scope step from this repo.

## GPU vs CPU-only

There are two separate templates, so the right one is picked directly from the Add Container Template dropdown — no post-install editing needed:

- **`modelship-cuda`** — GPU image (`ghcr.io/modelship-ai/modelship:latest-cuda`), needs the NVIDIA Driver plugin and NVIDIA Container Toolkit set up on your Unraid box. `Extra Parameters` includes `--runtime=nvidia`. (Plain `:latest`, with no suffix, is modelship's thin control/coordinator image — no torch/vllm — and will report 0 GPU/CPU capacity; don't use it for this template.)
- **`modelship-cpu`** — CPU-only image (`ghcr.io/modelship-ai/modelship:latest-cpu`), works on any box (amd64 or arm64, including Apple Silicon hosts), no GPU or NVIDIA Container Toolkit needed. `Extra Parameters` has no `--runtime=nvidia`, and the `NVIDIA_VISIBLE_DEVICES`/`NVIDIA_DRIVER_CAPABILITIES` variables are dropped entirely.

**Why two templates instead of one with a tag picker:** Unraid templates do support a `<Branch>`/tag-selector mechanism for offering multiple image tags from one template, but it only swaps the image tag — it can't conditionally change `Extra Parameters` or hide/show variables based on which tag is picked. `--runtime=nvidia` is a Docker *container-creation* flag, not something the image itself controls: it tells the Docker daemon to use a runtime named `nvidia`, which only exists if the NVIDIA Container Toolkit registered it. Without that toolkit, Docker refuses to even create the container (`unknown or invalid runtime name: nvidia`) — regardless of which image tag you picked. A single tag-picker template would need everyone selecting the CPU tag to also remember to manually delete `--runtime=nvidia` from Extra Parameters first, or the "CPU-only, no GPU needed" install would break immediately. Two templates avoid that footgun entirely — each one's `Extra Parameters` is correct by default for its own hardware target.

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

For GPU models (vLLM, Diffusers), multi-model stacks, and the full config reference, see [docs/model-configuration.md](https://github.com/modelship-ai/modelship/blob/main/docs/model-configuration.md) and the ready-made examples in [config/examples/](https://github.com/modelship-ai/modelship/tree/main/config/examples).

## `--shm-size` — read this before raising it

`--shm-size` (in **Extra Parameters**) is shared memory Ray uses to pass data between models fast. The template's default of `2g` is only sized for the single small quick-start model above.

There's no universal "correct" number — it scales with your box's RAM and what you're running:

- Bigger/more models, or a GPU stack → raise `--shm-size` to roughly 30% of the RAM you want to give modelship.
- Small Unraid boxes with little free RAM → keep it low; setting it far above what the host actually has doesn't cause problems immediately (the tmpfs is lazily backed), but if Ray/vLLM actually try to use that much, you'll hit real out-of-memory pressure instead of a clean startup error.

## What's exposed vs not

This template keeps the Config surface intentionally small — enough to get a working single-container deployment without needing to understand Ray, state stores, or gateway replicas:

- **Always visible**: API port, models.yaml path, model cache path, `HF_TOKEN`.
- **Advanced**: metrics port + toggle, `MSHIP_API_KEYS`, `MSHIP_LOG_LEVEL`, and (GPU template only) `NVIDIA_VISIBLE_DEVICES`/`NVIDIA_DRIVER_CAPABILITIES`.
- **Not exposed** (production/multi-node knobs that don't matter for a single Unraid container — defaults are fine, and you can still add any of these manually as extra Variables if you need them): gateway name/replicas/concurrency, state store (Redis/file), OpenTelemetry export, syslog log target/format, existing-Ray-cluster attach, Ray session pruning, preflight toggle, request body size limit, Ray head CPU/GPU pinning, Ray dashboard. Plugin backends (Kokoro ONNX, Orpheus, whisper.cpp) need no container-level config at all — they're pulled in automatically based on what your `models.yaml` references.

## Troubleshooting: edits to the container "don't stick"

If you edit the container in Unraid (add an env var, change the models.yaml path) and it looks like your change reverted — or a change you made earlier vanished when you changed something else — **check the running container before assuming the save failed.** The Unraid Docker **Edit** form caches aggressively and frequently shows stale values *after* you click Apply, even though the write already landed on disk (`/boot/config/plugins/dockerMan/templates-user/my-modelship-cuda.xml`) and on the live container.

Ground truth is `docker inspect`, which reads the actual running process, not the GUI's cache:

```sh
docker inspect modelship --format '{{range .Config.Env}}{{println .}}{{end}}'
```

If that shows your new value, the edit worked — just hard-refresh the Edit page (**Ctrl+Shift+R**) or close and reopen the Docker tab to clear the stale form.

Two things that *do* cause genuine reverts, worth ruling out if `docker inspect` disagrees with what you set:

- **Installed via Community Applications "Private" templates?** With CA's update check enabled, CA can re-merge the source template back over your container — bringing back removed variables and default paths while keeping your custom additions (the tell-tale "the var I deleted came back, but my other edits stayed" pattern). If you hit this, reinstall as a plain Docker container ([Option A](#installing) above): a plain-docker container with an empty `<TemplateURL>` and no CA source has nothing that can silently rewrite your edits.
- **Leftover `my-*.xml` files** in `templates-user/` from an earlier install can shadow the container you're editing. Keep only the `my-<name>.xml` for containers that actually exist (`docker ps -a`).

Note: `community.applications/private/<repo>/*.xml` is an install-*from* source (a store listing); `templates-user/my-<name>.xml` is the live container's config (what Edit reads and writes). They are not interchangeable — deleting the `my-` file orphans the running container's config rather than "moving" it.

## Icon

Both templates reference [`icon.png`](icon.png) in this repo. Unraid's Docker template `Icon` field doesn't render SVG (it falls back to a placeholder), so `icon.svg` is kept only as the source design and rasterized to `icon.png` for the templates.

## License

Apache 2.0, matching [modelship](https://github.com/modelship-ai/modelship).

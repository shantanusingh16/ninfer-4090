# Fork comparison: shantanusingh16/ninfer-4090 vs upstream

Date: 2026-08-18. Our fork: `shantanusingh16/ninfer-4090`, branch `rtx4090-port`.
Base: `sergiuszm/ninfer-4090`, branch `rtx4090-port` at `b6172f24`
(`feat(serve): report vision modality in models payloads`).

The upstream branch already carries the full `sm_89` RTX 4090 port — the Ada-retuned
attention prefill, E8 4-bit KV quantization at the native 262,144-token context, MTP3
speculative decoding, vision, and the llama.cpp-compatible `/metrics` and `/slots`
endpoints. This fork takes that exactly as-is and adds two things the unified
deployment needs to run alongside llama.cpp through llama-swap: per-request
performance telemetry the proxy can surface, and a build/run path that shares one
container with the rest of the stack.

## Commit delta over upstream

| Commit | Change |
|---|---|
| `41c67b9e` | `feat(serve): emit llama.cpp-compatible timings for proxy stats` |
| `9a60b206` | `feat(docker): unified llama-swap image + shared-volume model download` |

## 1. llama.cpp-compatible `timings` in responses (`41c67b9e`)

Upstream ninfer-serve writes the OpenAI `usage` token counts but **no** `timings`
object, so a proxy that derives per-request rates from the backend response (as
llama-swap does for llama.cpp and vLLM) reports Prefill/Decode speed, MTP draft
acceptance, and cache hits as unknown/empty for ninfer-served models.

This fork serializes the phase timing the engine already measures into a
llama.cpp-compatible top-level `timings` block, on:

- non-streaming `/v1/chat/completions` responses (and the tool-call variant),
- the streaming final chunk (with finish_reason), and
- the streaming `usage` chunk (when `stream_options.include_usage` is set).

Fields mirror llama.cpp's server contract so existing parsers read ninfer unchanged:
`prompt_n`, `predicted_n`, `prompt_ms`, `predicted_ms`, `prompt_per_second`,
`predicted_per_second`, `ttft_ms`, `cache_n`, and MTP `draft_n`/`draft_n_accepted`.

Where the data comes from: `GenerationMetrics` already tracked `prefill_seconds`,
`decode_seconds`, `ttft_seconds`, `prefix_cache_hit_tokens`,
`speculative_draft_tokens`, and `speculative_accepted_tokens` — no new engine
instrumentation, only serialization.

```json
"timings": {
  "prompt_n": 57, "predicted_n": 22,
  "prompt_ms": 112.77, "predicted_ms": 137.63,
  "prompt_per_second": 505.46, "predicted_per_second": 159.85,
  "ttft_ms": 113.08,
  "cache_n": 55,
  "draft_n": 18, "draft_n_accepted": 18
}
```

### Measured effect (llama-swap `/api/metrics/activity`, same 4090)

| Metric | Before | After |
|---|---:|---:|
| Prefill (prompt/s) | unknown | 505 / 2302 (cache hit) |
| Decode (tok/s) | unknown | 159.9 |
| Draft accepted / drafted | - | 18 / 18 |
| Cache tokens | - | 55 on prefix reuse |
| Duration | wall only | phase-derived |

Streaming and non-streaming both populate; this is the observable contract the
llama-swap dashboard reads.

## 2. Unified llama-swap image + shared-volume model download (`9a60b206`)

**`Dockerfile.llamaswap`.** Upstream's `Dockerfile` builds ninfer against CUDA 13.1
(`libcudart.so.13`), which the `ghcr.io/mostlygeek/llama-swap:unified-cuda` runtime
cannot load (it is CUDA 12.9, `libcudart.so.12`). This fork adds a second
Dockerfile that builds the engine against CUDA 12.9.2 (docs allow "12.8 or newer")
in a `nvidia/cuda:12.9.2-devel-ubuntu24.04` stage and copies
`ninfer`/`ninfer-serve` into the unified-cuda runtime stage — so one container
serves llama-swap, llama.cpp (`llama-server`), `ik-llama-server`, and ninfer
together, and llama-swap routes to `ninfer-serve` by model name. Upstream's
standalone `Dockerfile` (CUDA 13.1) is untouched for the single-engine path.

The build links `libcudart.so.12` and the non-CUDA libs already present in the
unified image (`libavcodec.so.60`, `libcurl.so.4`, ...): `ldd` on the deployed
binary shows zero `not found`.

**`scripts/download-qwen38.sh`.** The default `model_dir` moves from the repo-local
`models/` to the shared volume (`/run/media/.../Models/neroued/Qwen3.8-27B-NInfer`)
that the sibling llama.cpp models also live under, so the artifact is visible inside
the unified container at `/models/neroued/Qwen3.8-27B-NInfer/`. The `NINFER_MODEL_DIR`
override and the script's existing `-C -` resume behavior are unchanged.

## What deliberately did NOT change

- Upstream `Dockerfile` (CUDA 13.1 standalone path), `docs/*`, engine/runtime code,
  ops/kernels, and the `/metrics`, `/slots`, vision-modality, and E8 KV behavior are
  untouched — `git diff b6172f24..HEAD -- src/ runtime/` is empty apart from the
  `serve/` serialization added in `41c67b9e`.
- No model-id aliasing, queue, or admission defaults were changed in the engine; the
  `/v1/models` `vision` modality and `context_window` fields come from upstream
  `b6172f24`.

## Deployment wiring (outside this repo)

The run command (image `ninfer-swap:unified`, detached `--name llama-swap`, port
8080, `/models` mount) and the llama-swap config entry
(`qwen3.8-27b-ninfer`, full 262K native context, `rk4v4-e8`, MTP3, 10-minute
admission deadline) live in the user's `/home/shantanu/tools/llm_setup` tree, not
in this repository. This fork's repository contribution is limited to the two
commits above; the config and launcher are documented in the deployment notes.

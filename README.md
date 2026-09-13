# Qwen-Flash-Colab-G4

[Open the notebook in Google Colab](https://colab.research.google.com/github/fabioamigo/Qwen-Flash-Colab-G4/blob/main/Qwen38_FlashNext_NVFP4_FRSpec_Cloudflare.ipynb)

Run **RadixArk/Qwen3.8-Flash-Next-NVFP4** with the **Pennyroyal SGLang FR-Spec launcher** and expose a temporary, authenticated OpenAI-compatible API through a Cloudflare Quick Tunnel.

This repository provides a Colab bootstrap notebook. The inference implementation, kernels, model weights, chat template, and speculative decoding work belong to the upstream projects credited below.

## Quick start

1. Download `Qwen38_FlashNext_NVFP4_FRSpec_Cloudflare.ipynb` from this repository and upload it to [Google Colab](https://colab.research.google.com/). Alternatively, use Colab's GitHub tab with this repository's URL.
2. Select a **G4 runtime with an NVIDIA RTX PRO 6000 Blackwell Server Edition, 96 GB VRAM, 48 logical CPUs, and high host RAM**. The notebook checks the actual GPU, rather than trusting the runtime label. T4, L4, A100, and H100 are not substitutes for this SM120 recipe.
3. Run all cells. The first code cell optionally accepts a Hugging Face read token through a hidden prompt. Press Enter to continue anonymously. You can create a token at [Hugging Face Access Tokens](https://huggingface.co/settings/tokens).
4. **Allow more than 30 minutes for the server to become ready on a fresh Google Colab G4 instance.** Installation, checkpoint download, model loading, kernel compilation, autotuning, and warmup all contribute to the wait. Logs and periodic progress messages remain visible.
5. The publication cell reports success only after checking the public model list, rejection of missing/incorrect API keys, and a short generation through the public URL.
6. Copy the printed base URL, API key, and curl example into your client.
7. Use the final **Encerrar API** button to stop both processes. Running all cells only displays the controls; it does not automatically shut down the API.

The notebook's explanatory text and progress messages are in Portuguese. This README provides the English usage guide.

## Requirements

| Resource | Requirement or behavior |
| --- | --- |
| GPU | Blackwell SM120, approximately 96 GB VRAM |
| CPU profile | Tuned for a dedicated 48-logical-CPU G4 runtime |
| Host OS | Ubuntu 22.04 or 24.04, Linux x86-64 |
| Host RAM | Preflight budgets 95.68 GiB available: 47.68 GiB PLE + 32 GiB HiCache + 16 GiB margin; this is a conservative operational budget, not a measured model minimum |
| Model storage | Approximately 125.91 GiB across 206 indexed weight files, plus selected metadata |
| Additional storage | Toolchain, Python dependencies, build artifacts and caches; download preflight separately reserves 12 GiB beyond remaining selected files |
| Driver | R580/R590/R595 use `cuda-compat-13-3`; R610+ uses the native driver path; actual initialization and PTX JIT are checked |
| Internet | GitHub, Ubuntu/NVIDIA package repositories, Python indexes, Hugging Face/Xet, and Cloudflare |

The notebook cannot upgrade the host's NVIDIA kernel driver. Forward compatibility is subject to device/platform support, and the early CUDA test fails visibly if it is not supported.

## What the notebook does

There are seven code cells:

1. Optional Hugging Face token input.
2. Configuration, hardware checks, and process helpers.
3. Native toolchain, Python, SGLang, NIXL POSIX, and CUDA/PTX checks.
4. Selective checkpoint download and validation.
5. FR-Spec server startup and local API tests.
6. Cloudflare Quick Tunnel setup and public API tests.
7. Log and shutdown controls.

Installation uses a separate **uv-managed Python 3.12.13 interpreter**, without a virtual environment and without replacing the Colab kernel's PyTorch. It installs CUDA 13.3 user-space components, GCC 15, Rust/Cargo, the pinned SGLang source, and NIXL with the POSIX plugin.

The APT bootstrap consolidates duplicate CUDA sources before the first `apt-get update`, keeping backups of changed source files. SGLang alone uses non-isolated Python building; its dependencies retain build isolation so declared backends such as `wheel-stub` are installed correctly.

## Model and serving configuration

| Setting | Value |
| --- | --- |
| Hugging Face model | `RadixArk/Qwen3.8-Flash-Next-NVFP4` |
| Public API model name | `pennyroyal` |
| Context window | 524,288 tokens, including prompt and generated tokens |
| Context extension request | YaRN factor 2 from 262,144 positions |
| GPU token pool | 824,384 tokens shared across requests |
| Maximum concurrent requests | 4, subject to resource availability |
| KV dtype | FP8 E4M3 |
| Speculation | Native NEXTN MTP with the bundled FR-Spec 64K token map |
| MTP download | Included in the target checkpoint; no separate draft repository |
| Prefix caching | HiCache in host RAM and NIXL POSIX on instance-local disk |
| PLE backend | RAM |
| Online FP8 option | Disabled by default |
| Media preprocessing | CPU |
| Default reasoning settings | Thinking enabled, preserve thinking enabled, medium effort |

The official launcher is retained. An executable wrapper adds loopback binding, API-key authentication, and the configured weight-loader worker count.

## Aggressive parallelism

| Operation | Default |
| --- | ---: |
| Concurrent model downloads | 32 files |
| Build jobs: Ninja, CMake, Cargo, Make | 48 |
| FlashInfer NVCC internal threads | 4 per compiler process |
| Weight-loader workers | 24 |

`HF_XET_HIGH_PERFORMANCE=1` enables Xet's high-performance transfer mode. A Hugging Face token authenticates requests; it does not guarantee higher transfer bandwidth.

`MAX_JOBS=48` is the effective Ninja limit used by the inspected FlashInfer implementation. Cargo can build independent dependencies concurrently; each rustc invocation is not configured to use 48 internal threads. Nested compiler parallelism can exceed 48 runnable threads and increase peak RAM usage. Serial work, dependency chains, storage bandwidth, and GPU autotuning can still leave CPUs idle.

These settings are configurable in the configuration cell. They are aggressive defaults, not a demonstrated speedup. Changing compiler flags can invalidate some compiled caches.

## API usage

Use the values printed by the notebook:

```bash
export OPENAI_BASE_URL='https://YOUR-TEMPORARY-HOST.trycloudflare.com/v1'
export OPENAI_API_KEY='YOUR-GENERATED-KEY'

curl --fail-with-body --max-time 120 "$OPENAI_BASE_URL/chat/completions" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "pennyroyal",
    "messages": [{"role": "user", "content": "Reply with READY."}],
    "max_tokens": 32,
    "stream": false,
    "chat_template_kwargs": {"enable_thinking": false}
  }'
```

This shell example is for Bash/Zsh on Linux/macOS; on Windows, use WSL or adapt the environment-variable and quoting syntax for PowerShell.

**Cloudflare Quick Tunnels do not officially support SSE. Use `stream: false`.** Clients that require streamed chat completions need a different supported transport, such as a configured named Cloudflare Tunnel. Long non-streaming requests are also subject to Cloudflare timeouts.

The API key is generated with `secrets.token_urlsafe(32)` whenever the server startup cell runs. It has no independent time-based expiry: stopping or replacing that server ends access to that running instance. The URL is temporary and depends on the tunnel process remaining alive.

## Coding-agent examples

See [Pi and OpenCode setup](examples/README.md) for configuration files, environment-based API credentials, and a streaming connection check:

- [Pi models.json](examples/pi/models.json)
- [OpenCode opencode.json](examples/opencode/opencode.json)

Both clients require an SSE-capable route to the server. The notebook's default Quick Tunnel does not officially support SSE; the guide explains this prerequisite. The examples use `pennyroyal`, a 524,288-token context, and a configurable 131,072-token output ceiling. Both expose `off`, `medium`, and `xhigh` thinking levels and explicitly set temperature 1.0, top-p 0.95, top-k 20, presence penalty 0.0, and repetition penalty 1.0.

## Validation status and startup expectations

An earlier configuration was observed to complete local startup and return a short answer on the user's Colab GPU. Its log reported `context_len=524288` and an allocated 824,384-token KV pool. That is **not** a validation of output quality at 512K, all multimodal features, public streaming, or full cache restoration.

The observed startup log included warnings about the derived 262,144-token context and RoPE configuration validation. The requested YaRN override was present in server arguments, but long-context correctness still requires dedicated testing.

That earlier logged startup took approximately 30 minutes from the first timestamp to application readiness: about 16 minutes loading target/MTP weights and 13 minutes in the initial FlashInfer autotune context, which can also trigger compilation. Do not interpret all of that interval as CPU compilation.

The repository owner has reported successful startup of the published version with the 48-job / 32-download / 24-loader profile. Plan for **more than 30 minutes before the server is ready on a fresh Google Colab G4 instance**; the total varies with downloads, compilation, and cache availability.

## Code-generation performance and monitoring

**Expected code-generation throughput on the Google Colab G4 runtime is more than 200 tokens per second**, based on the repository owner's reported expectation for this setup. Actual throughput varies with the workload, context length, concurrency, and speculative decoding acceptance; this is not a guaranteed minimum.

To monitor the generation throughput reported by SGLang, open a **terminal inside the running Colab instance** and run:

```bash
watch -n 1 "tail -n 20 /content/pennyroyal-colab/logs/server.log | grep -oP 'gen throughput \(token/s\): \d+\.\d+'"
```

The command refreshes every second and extracts throughput values from the last 20 log lines. It may show multiple samples or no output if those lines contain no matching generation metric. Press Ctrl+C to stop monitoring. This is the server-reported generation metric, not end-to-end client latency or prompt-processing speed.

## Persistence and troubleshooting

- Files live under `/content/pennyroyal-colab`. Restarting only the server keeps model files and caches; losing the VM loses them.
- Logs are under `/content/pennyroyal-colab/logs`.
- Re-running the server cell replaces this notebook's server/tunnel processes and rotates the API key.
- Keep runtime paths and dependency versions stable to improve cache reuse.
- Do not publish executed notebook outputs: the endpoint cell prints the temporary API credentials for your use.
- If a token is invalid or lacks access, correct it in the initial token cell before retrying the download.
- If parallel compilation exhausts RAM, lower `BUILD_JOBS` and/or `NVCC_THREADS`; weight-loader memory pressure is controlled separately by `WEIGHT_LOAD_WORKERS`.
- The notebook does not bypass Colab session limits. It does not keep the VM alive indefinitely.

## Pinned upstream identities

| Component | Identity |
| --- | --- |
| Pennyroyal tag | `pennyroyal-v2.5.0` |
| Pennyroyal checkout | `2c675da096939cb01102f8f4871bda3db55f7f28` |
| Model revision | `7b719225242aacd3dbd3f9407468c2ee9a9d2594` |
| NIXL commit | `aecbc3846d92c34c7507a58d776e1fda50ff4fba` |

This is not a complete dependency lock. Some transitive packages and the Cloudflare release are resolved during installation.

## Credits and inspiration

### Direct recipe and implementation

- **[jpezzulli/sglang-rtxpro6000](https://github.com/jpezzulli/sglang-rtxpro6000)** — Pennyroyal runtime, native build instructions, RTX PRO 6000 integration, FR-Spec launcher, cache configuration, and pinned template. This is the primary recipe and inspiration for the notebook. See [BUILD.md](https://github.com/jpezzulli/sglang-rtxpro6000/blob/pennyroyal-v2.5.0/BUILD.md) and the [FR-Spec launcher](https://github.com/jpezzulli/sglang-rtxpro6000/blob/pennyroyal-v2.5.0/configs/pennyroyal/serve-flash-next-frspec.sh).
- **[sgl-project/sglang](https://github.com/sgl-project/sglang)** — upstream inference-serving framework on which Pennyroyal is based.
- **[gabrielolympie/sglang-flashnext-sm120](https://github.com/gabrielolympie/sglang-flashnext-sm120)** — FR-Spec reduced-draft-vocabulary work credited by Pennyroyal.
- **[RadixArk/Qwen3.8-Flash-Next-NVFP4](https://huggingface.co/RadixArk/Qwen3.8-Flash-Next-NVFP4)** — the public NVFP4 checkpoint used by this notebook. The underlying model and its authors are identified in the model card.
- **Froggeric** — v22.5 chat template bundled and checksum-pinned by Pennyroyal; see the upstream [template attribution](https://github.com/jpezzulli/sglang-rtxpro6000/blob/pennyroyal-v2.5.0/configs/pennyroyal/templates/README.md).

### Runtime and tooling

- **[flashinfer-ai/flashinfer](https://github.com/flashinfer-ai/flashinfer)** — GPU kernels, JIT infrastructure, and autotuning.
- **[ai-dynamo/nixl](https://github.com/ai-dynamo/nixl)** — NIXL and its POSIX storage plugin for the hierarchical cache.
- **[huggingface/huggingface_hub](https://github.com/huggingface/huggingface_hub)** and **[huggingface/xet-core](https://github.com/huggingface/xet-core)** — checkpoint metadata, authenticated downloads, and Xet transfers.
- **[astral-sh/uv](https://github.com/astral-sh/uv)** — Python interpreter and dependency installation.
- **[cloudflare/cloudflared](https://github.com/cloudflare/cloudflared)** — temporary public tunnel.
- **[pytorch/pytorch](https://github.com/pytorch/pytorch)** and the NVIDIA CUDA toolchain — tensor execution and native compilation.

### Additional upstream influences, not enabled as the default recipe here

Pennyroyal also credits **[mratsim/sglang-qwen38fn-sm120-turbo](https://github.com/mratsim/sglang-qwen38fn-sm120-turbo)** for inspiration behind its optional online-FP8 implementation, and **[garnermccloud/sglang-ssd-stream](https://github.com/garnermccloud/sglang-ssd-stream)** for its optional NVMe PLE reader. The notebook defaults to online FP8 off and RAM-backed PLE; it does not claim to implement or enable those optional projects independently.

## License

This repository is distributed under the [Apache License 2.0](LICENSE).

## Third-party licenses

Model weights and third-party source code are downloaded from upstream and are not bundled here. Each dependency, model, template, and upstream component retains its own license and attribution requirements. Consult the linked repositories and model card before reuse or redistribution. This notebook is an integration of those projects, not an official release or endorsement by their maintainers.

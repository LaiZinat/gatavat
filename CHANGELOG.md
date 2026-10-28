# Changelog

## Unreleased

### Added

- `llmctl oc [profile]` (alias `llmctl code`) starts opencode with
  `OPENCODE_CONFIG=<profiles_dir>/<profile>.json`. Use `-s <id>` to start models first and wait
  until they're ready, `--list` to list profiles, and `--` to pass arguments to opencode. New
  global settings: `opencode_bin`, `opencode_profiles_dir`, `opencode_default_profile`.
- Example model `devstral2-123b` (Devstral 2 123B Instruct, Q3_K_S, port 8106, 16k context
  with q8_0 KV cache) and `devstral-small2-24b` (Devstral Small 2 24B Instruct, Q8_0, port 8107,
  64k context with q8_0 KV cache). Models you add now start at port 8108.
- **opencode reminders:** the web UI shows a banner when `opencode.json` no longer matches your
  models (e.g. after changing a context size), with a one-click write, followed by a reminder to
  restart opencode. `llmctl set/add/reset`, `llmctl status` and `llmctl opencode --write` print
  the same hints.
- **Context size dropdown** in the web UI with the standard sizes (2k to 256k). Each option shows
  its KV-cache memory and the change from the current setting, and flags sizes above the trained
  context. "Other…" accepts any value.

- `llmctl ui stop | restart | status` and `llmctl ui -d` (background). `llmctl ui` opens the
  running UI instead of failing with "Address already in use". `llmctl status` shows the UI.

- Gateway fix for strict chat templates (Mistral/Devstral): system messages are merged into one
  and consecutive same-role messages are joined, so opencode's requests no longer fail with
  "conversation roles must alternate". Auto-detected from the GGUF; new setting `fix_roles`.

- **Sharing with other machines:** `llmctl share` (`--enable`, `--host`, `--allow`, `--new-key`,
  `--disable`) exposes the gateway with an API key and prints addresses, the key, a curl test,
  OpenAI environment variables and an opencode provider for the other machine. New global
  settings `gateway_api_key` and `gateway_allow`, also in the UI's Settings. The gateway log shows
  each request's client address.
- Guide [docs/TAILSCALE.md](docs/TAILSCALE.md) for sharing models over Tailscale, for host and guest.
  `llmctl share` lists this machine's Tailscale name and address.
- `llmctl logs gateway` shows the gateway log.

### Changed

- The gateway refuses requests from other machines unless `gateway_api_key` is set. Before, setting
  `gateway_host=0.0.0.0` let anyone on the network use it. Requests from this machine are unchanged,
  except ones relayed by a local proxy such as `tailscale serve` (recognized by their proxy headers).

### Fixed

- KV-cache estimate for Qwen3-Next (`qwen3next`) models was 4× too high: only every 4th layer
  has full attention, but those GGUFs don't store the interval.
- Endpoint links in the web UI were unreadable in dark mode.
- The gateway was reported as stopped while running when Python is a macOS framework build
  (e.g. Homebrew), so every model start tried to launch a second gateway ("something is already
  listening on port 8100").

## 0.1.0 (2026-10-22)

First public release.

### Features

- **Model management** for llama.cpp's `llama-server`: configure models once (path, alias,
  host, port, context, GPU layers, threads, parallel slots, batch sizes, flash attention,
  KV-cache types, Jinja templates, embeddings/pooling, sampling defaults, extra flags) and
  start, stop, restart, or run them in the foreground. Several models can run at once.
- **CLI and web UI sharing one state.** Models run detached, and either front end can control
  models started by the other.
- **One-off overrides** at start (`--ctx`, `--port`, `-s key=value`), optionally saved with
  `--save`.
- **"Restart to apply" detection** when a running model's saved settings changed.
- **Split GGUF support:** `X.gguf` resolves to `X-00001-of-N.gguf` next to it or in a folder `X`.
- **Model specs from the GGUF header** (`llmctl info`): architecture, quantization, trained
  context, layers, heads, experts, chat template and thinking support, plus a KV-cache and total
  memory estimate for the configured context. Warns about context beyond the trained length,
  missing split parts, and mismatched settings.
- **Thinking switch** for hybrid reasoning models, shown only when the model supports it. It
  uses whichever mechanism the installed llama-server build supports.
- **Chat test** in CLI and UI with streaming, reasoning display, per-request thinking switch and
  tokens/s. Embedding models get a dimension/norm/cosine-similarity test.
- **Gateway:** one OpenAI-compatible endpoint (`:8100/v1`) that routes by model name, with
  clear errors for unknown or stopped models. It starts automatically with the first model.
- **opencode export** (`llmctl opencode --write`): one `llama-local` provider via the gateway
  (or one provider per model), with context and output limits. The existing file is backed up,
  other content is kept, and model references are migrated.
- **Logs** per model (rotated at 20 MB), viewable with `llmctl logs -f` or live in the UI.
- **Safe defaults:** everything listens on `127.0.0.1`; the UI rejects cross-site and
  DNS-rebinding requests.

### Requirements

macOS (tested on Apple Silicon), Python 3.8+, a recent `llama-server`, GGUF models. Linux is
expected to work but is untested; Windows is not supported.

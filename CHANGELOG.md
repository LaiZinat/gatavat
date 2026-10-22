# Changelog

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

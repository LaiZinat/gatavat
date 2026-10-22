# Usage and configuration reference

- [Everyday commands](#everyday-commands)
- [Model settings](#model-settings)
- [Global settings](#global-settings)
- [Model specs and memory](#model-specs-and-memory)
- [Chat test](#chat-test)
- [Gateway](#gateway)
- [Web UI](#web-ui)
- [Built-in example models](#built-in-example-models)
- [Updating](#updating)
- [Files](#files)
- [Troubleshooting](#troubleshooting)

## Everyday commands

```sh
llmctl                          # status of all models + gateway (same as: llmctl status / ls / ps)
llmctl ui [--port N] [--no-open]  # web UI, default http://127.0.0.1:8090
llmctl start                    # start the models marked autostart
llmctl start a b -w             # start several, wait until loaded
llmctl start --all
llmctl stop a | --all
llmctl restart a                # apply changed settings
llmctl logs a [-n 200] [-f]     # server log
llmctl run a                    # foreground, for debugging (Ctrl-C stops it)
llmctl cmd a                    # print the exact llama-server command
```

Models run detached in their own process group. Their pid and command are stored in
`~/.config/llmctl/run/<id>.json`, which is how the CLI and the UI share state.

## Model settings

```sh
llmctl show a                           # all settings, grouped, plus the command line
llmctl set a ctx_size=131072 flash_attn=on
llmctl set a parallel=                  # empty value = unset (use llama-server's default)
llmctl start a --ctx 65536              # one-off override; --save keeps it
llmctl start a -s temp=0.2 -s top_k=40  # any setting as a one-off
llmctl add b --from a model=~/Models/x.gguf
llmctl rm b
llmctl reset [a ...]                    # restore built-in defaults
```

| setting | llama-server flag | notes |
|---|---|---|
| name | | display name |
| model | `-m` | path to the `.gguf`; `~`, `$VAR`, `${VAR:-default}` allowed, split models supported (see below) |
| alias | `--alias` | model name on `/v1/models`; also the name the gateway and opencode use |
| host | `--host` | `127.0.0.1` = this machine only, `0.0.0.0` = whole network |
| port | `--port` | |
| ctx_size | `--ctx-size` | context window in tokens |
| autostart | | started by `llmctl start` with no ids |
| ngl | `-ngl` | layers on the GPU, `999` = all |
| threads | `--threads` | number or `auto` (all cores) |
| parallel | `--parallel` | request slots |
| batch_size / ubatch_size | `--batch-size` / `--ubatch-size` | |
| flash_attn | `--flash-attn` | `on` / `off` / `auto` |
| cache_type_k / cache_type_v | `--cache-type-k/v` | e.g. `q8_0` roughly halves KV memory (V needs flash attention) |
| jinja | `--jinja` | use the model's chat template; needed for tool calling |
| thinking | see below | `on` / `off` / blank = model default |
| embedding | `--embedding` | serve embeddings only |
| pooling | `--pooling` | embedding pooling, e.g. `last` |
| temp, top_p, top_k, min_p, repeat_penalty | `--temp` ... | server-side sampling defaults; clients can override |
| extra_args | | any other flags, e.g. `--mlock --no-mmap` |
| opencode_provider | | blank = not exported to opencode (see [OPENCODE.md](OPENCODE.md)) |
| opencode_output | | max output tokens advertised to opencode |

**Split models.** Point `model` at the first part (`X-00001-of-00004.gguf`), or at `X.gguf` if
the parts are named `X-00001-of-N.gguf` and sit next to it or in a folder named `X`.
llama-server loads the remaining parts itself.

**Thinking.** For hybrid reasoning models (for example Qwen3.6), `thinking=off` disables
reasoning on the server. The setting only appears in the UI when the model's chat template has
an `enable_thinking` switch. llmctl checks your llama-server's `--help` and uses whichever
mechanism the build supports: `--reasoning-budget`, `--reasoning`, or `--chat-template-kwargs`.

**Restart to apply.** When a running model's saved settings differ from the ones it was started
with, status shows "config changed, restart to apply" and the UI shows a warning.

## Global settings

```sh
llmctl global                       # show all, plus which llama-server binary was found
llmctl global key=value ...
```

| key | default | meaning |
|---|---|---|
| llama_server | `${LLAMA_SERVER:-llama-server}` | binary name on PATH or full path. Also searched in `/opt/homebrew/bin`, `/usr/local/bin`, `~/.local/bin` |
| ui_host / ui_port | `127.0.0.1` / `8090` | web UI address |
| gateway_host / gateway_port | `127.0.0.1` / `8100` | gateway address |
| gateway_autostart | `true` | start the gateway whenever a model starts |
| opencode_path | `~/.config/opencode/opencode.json` | file written by `llmctl opencode --write` |
| opencode_mode | `gateway` | `gateway` = one provider for all models; `per-model` = one provider per model port |
| opencode_provider_id | `llama-local` | provider id in gateway mode |

## Model specs and memory

```sh
llmctl info [a ...] [--json]
```

Reads only the GGUF header (milliseconds, the model is not loaded) and shows:

- name, architecture, size label, quantization
- file size across all split parts, tensor count
- trained vs. configured context
- layers (and how many use attention, for hybrid models), embedding size, heads / KV heads, experts
- whether a chat template is included, and whether thinking can be switched
- **estimated KV-cache memory** for the configured context and cache type, and an estimated
  total (file + KV, without compute buffers)

It warns when `ctx_size` exceeds the trained context, split parts are missing, or `thinking` /
`jinja` don't fit the model. The UI shows the same on each card.

## Chat test

```sh
llmctl chat a                         # interactive
llmctl chat a "one-shot prompt" -t 0.2 -m 512
llmctl chat a --system "Be terse"
llmctl chat a "hi" --think off        # per-request thinking switch (hybrid models)
llmctl chat a "hi" --no-think         # only hides the reasoning output
llmctl chat emb "some text"           # embedding model: dim, norm, cosine vs. previous input
```

Interactive commands: `/clear`, `/system <text>`, `/think on|off|default`, `/exit`. Ctrl-C
stops a reply. Each answer ends with generation and prompt speed in tokens/s.

In the UI, every ready model has a **Chat** button (**Test** for embedding models) with a system
prompt, temperature, max tokens and a thinking switch where supported.

## Gateway

```sh
llmctl gateway [status|start|stop|restart]
```

A single OpenAI-compatible endpoint, `http://127.0.0.1:8100/v1`. Each `POST /v1/...` is
forwarded to the model named in the request's `model` field: alias or id, with or without a
`provider/` prefix. Responses, including streams, are passed through unchanged.

- `GET /v1/models` lists all configured models with their state and port.
- A request for a stopped model returns HTTP 503 with "start it with `llmctl start <id>`". Models
  are not loaded on demand.
- It starts with the first model (`gateway_autostart`) and keeps running after models stop. It
  uses almost no resources.
- The log is at `~/.config/llmctl/logs/_gateway.log`.

## Web UI

`llmctl ui` serves the UI on `127.0.0.1:8090` and opens the browser. Each model has a card with
status, endpoint, specs summary, Start / Restart / Stop / Logs / Chat, the context size and port,
and all other settings under "All settings & command". The header shows the gateway status,
**Start autostart models**, **Stop all**, the **opencode.json** preview and write, **Add model**,
and global **Settings**.

Closing the UI (Ctrl-C) does not stop models or the gateway.

## Built-in example models

The default config is the author's setup and doubles as an example. Paths are under `~/Models`;
override them in the config or with the listed environment variables.

| id | model | port | ctx | default path (env override) |
|---|---|---|---|---|
| qwen25-14b | Qwen2.5 14B Instruct Q4_K_M | 8101 | 8192 | `${QWEN_MODELS:-~/Models/LLM/Qwen2.5-14B-Instruct}/qwen2.5-14b-instruct-q4_k_m.gguf` |
| qwen25-32b | Qwen2.5 32B Instruct Q4_K_M | 8102 | 32768 | `~/Models/LLM/Qwen2.5-32B-Instruct/qwen2.5-32b-instruct-q4_k_m-00001-of-00005.gguf` |
| qwen3-coder | Qwen3-Coder-Next Q4_K_M | 8103 | 65536 | `${CODER_MODEL:-~/Models/LLM/Qwen3-Coder-Next/Qwen3-Coder-Next-Q4_K_M/Qwen3-Coder-Next-Q4_K_M-00001-of-00004.gguf}` |
| qwen36-35b | Qwen3.6-35B-A3B Q8_0 | 8104 | 32768 | `${OPENHANDS_MODEL:-~/Models/LLM/Qwen3.6-35B-A3B/Qwen3.6-35B-A3B-Q8_0.gguf}` |
| qwen3-embedding | Qwen3-Embedding-8B f16 | 8105 | 8192 | `${EMBEDDING_MODEL:-~/Models/Embedding/Qwen3-Embedding-8B/Qwen3-Embedding-8B-f16.gguf}` |

Ports follow 8100 + i; models you add get the next free port from 8106. The gateway is on 8100
and the UI on 8090.

## Updating

```sh
cd ~/Projects/gatavat && git pull   # wherever you cloned it
```

CLI commands always run the current code. Long-running llmctl processes need a restart to pick
up new code:

- **web UI:** Ctrl-C, then `llmctl ui`
- **gateway:** `llmctl gateway restart`

Running models don't need a restart; they are llama-server processes, not llmctl code. Your
config is kept. New settings appear with their defaults.

## Files

| path | contents |
|---|---|
| `~/.config/llmctl/config.json` | all settings. Set `LLMCTL_HOME` to use another folder |
| `~/.config/llmctl/logs/<id>.log` | server logs, rotated at 20 MB; `_gateway.log` for the gateway |
| `~/.config/llmctl/run/<id>.json` | pid and command of running servers |
| `~/.config/llmctl/server-flags.json` | cached `llama-server --help` flags |

## Troubleshooting

| symptom | cause / fix |
|---|---|
| "model file not found" | Wrong `model` path. `llmctl show <id>` prints the expanded path |
| "exited (see logs)" right after start | `llmctl logs <id>`. Common causes: `unknown model architecture` (llama.cpp too old, so update it), not enough memory, port in use |
| "'llama-server' not found on PATH" | `llmctl global llama_server=/full/path/to/llama-server` |
| "something is already listening on port N" | Another program uses the port. Change it with `llmctl set <id> port=...` |
| new UI features missing after an update | Restart the UI (and `llmctl gateway restart`) |
| CMake build errors after updating llama.cpp | `cmake -B build --fresh` (a stale cache can reference an old macOS SDK) |

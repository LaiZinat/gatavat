# Usage and configuration reference

- [Everyday commands](#everyday-commands)
- [Model settings](#model-settings)
- [Global settings](#global-settings)
- [Model specs and memory](#model-specs-and-memory)
- [Chat test](#chat-test)
- [Gateway](#gateway)
- [Sharing with other machines](#sharing-with-other-machines)
- [Web UI](#web-ui)
- [Built-in example models](#built-in-example-models)
- [Updating](#updating)
- [Files](#files)
- [Troubleshooting](#troubleshooting)

## Everyday commands

```sh
gatavat                          # status of all models + gateway (same as: gatavat status / ls / ps)
gatavat ui [--port N] [--no-open]  # web UI, default http://127.0.0.1:8090 (opens it if already running)
gatavat ui -d                    # run the UI in the background, no terminal needed
gatavat ui stop | restart | status
gatavat start                    # start the models marked autostart
gatavat start a b -w             # start several, wait until loaded
gatavat start --all
gatavat stop a | --all
gatavat restart a                # apply changed settings
gatavat logs a [-n 200] [-f]     # server log (`gatavat logs gateway` for the gateway)
gatavat run a                    # foreground, for debugging (Ctrl-C stops it)
gatavat cmd a                    # print the exact llama-server command
```

Models run detached in their own process group. Their pid and command are stored in
`~/.config/gatavat/run/<id>.json`, which is how the CLI and the UI share state.

## Model settings

```sh
gatavat show a                           # all settings, grouped, plus the command line
gatavat set a ctx_size=131072 flash_attn=on
gatavat set a parallel=                  # empty value = unset (use llama-server's default)
gatavat start a --ctx 65536              # one-off override; --save keeps it
gatavat start a -s temp=0.2 -s top_k=40  # any setting as a one-off
gatavat add b --from a model=~/Models/x.gguf
gatavat rm b
gatavat reset [a ...]                    # restore built-in defaults
```

| setting | llama-server flag | notes |
|---|---|---|
| name | | display name |
| model | `-m` | path to the `.gguf`; `~`, `$VAR`, `${VAR:-default}` allowed, split models supported (see below) |
| alias | `--alias` | model name on `/v1/models`; also the name the gateway and opencode use |
| host | `--host` | `127.0.0.1` = this machine only, `0.0.0.0` = whole network |
| port | `--port` | |
| ctx_size | `--ctx-size` | context window in tokens |
| autostart | | started by `gatavat start` with no ids |
| ngl | `-ngl` | layers on the GPU, `999` = all |
| threads | `--threads` | number or `auto` (all cores) |
| parallel | `--parallel` | request slots |
| batch_size / ubatch_size | `--batch-size` / `--ubatch-size` | |
| flash_attn | `--flash-attn` | `on` / `off` / `auto` |
| cache_type_k / cache_type_v | `--cache-type-k/v` | e.g. `q8_0` roughly halves KV memory (V needs flash attention) |
| jinja | `--jinja` | use the model's chat template; needed for tool calling |
| thinking | see below | `on` / `off` / blank = model default |
| fix_roles | | gateway merges system messages and consecutive same-role messages for strict chat templates; blank = auto-detect (see below) |
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
an `enable_thinking` switch. gatavat checks your llama-server's `--help` and uses whichever
mechanism the build supports: `--reasoning-budget`, `--reasoning`, or `--chat-template-kwargs`.

**Strict chat templates.** Some templates (Mistral, Devstral) reject a conversation unless it is
exactly one system message followed by alternating user and assistant turns ("conversation roles
must alternate"). Clients such as opencode sometimes send two system messages or two user
messages in a row. For these models the gateway merges them before forwarding. It detects strict
templates from the GGUF automatically; override with `fix_roles=on|off`. This only applies to
requests through the gateway, not to direct requests to the model's own port.

**Restart to apply.** When a running model's saved settings differ from the ones it was started
with, status shows "config changed, restart to apply" and the UI shows a warning.

## Global settings

```sh
gatavat global                       # show all, plus which llama-server binary was found
gatavat global key=value ...
```

| key | default | meaning |
|---|---|---|
| llama_server | `${LLAMA_SERVER:-llama-server}` | binary name on PATH or full path. Also searched in `/opt/homebrew/bin`, `/usr/local/bin`, `~/.local/bin` |
| ui_host / ui_port | `127.0.0.1` / `8090` | web UI address |
| gateway_host / gateway_port | `127.0.0.1` / `8100` | gateway address |
| gateway_autostart | `true` | start the gateway whenever a model starts |
| gateway_api_key | (blank) | key other machines must send (`Authorization: Bearer <key>`). Blank = other machines are refused |
| gateway_allow | (blank) | IPs or networks allowed from other machines, e.g. `192.168.1.0/24, 100.64.0.0/10`. Blank = any address with the key |
| opencode_path | `~/.config/opencode/opencode.json` | file written by `gatavat opencode --write` |
| opencode_mode | `gateway` | `gateway` = one provider for all models; `per-model` = one provider per model port |
| opencode_provider_id | `llama-local` | provider id in gateway mode |
| opencode_bin | `opencode` | opencode binary used by `gatavat oc` |
| opencode_profiles_dir | `~/.config/opencode/profiles` | where `gatavat oc <name>` looks for `<name>.json` |
| opencode_default_profile | (blank) | profile used by a bare `gatavat oc`; blank = opencode's normal config |

## Model specs and memory

```sh
gatavat info [a ...] [--json]
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
gatavat chat a                         # interactive
gatavat chat a "one-shot prompt" -t 0.2 -m 512
gatavat chat a --system "Be terse"
gatavat chat a "hi" --think off        # per-request thinking switch (hybrid models)
gatavat chat a "hi" --no-think         # only hides the reasoning output
gatavat chat emb "some text"           # embedding model: dim, norm, cosine vs. previous input
```

Interactive commands: `/clear`, `/system <text>`, `/think on|off|default`, `/exit`. Ctrl-C
stops a reply. Each answer ends with generation and prompt speed in tokens/s.

In the UI, every ready model has a **Chat** button (**Test** for embedding models) with a system
prompt, temperature, max tokens and a thinking switch where supported.

## Gateway

```sh
gatavat gateway [status|start|stop|restart]
```

A single OpenAI-compatible endpoint, `http://127.0.0.1:8100/v1`. Each `POST /v1/...` is
forwarded to the model named in the request's `model` field: alias or id, with or without a
`provider/` prefix. Responses, including streams, are passed through unchanged.

- `GET /v1/models` lists all configured models with their state and port.
- A request for a stopped model returns HTTP 503 with "start it with `gatavat start <id>`". Models
  are not loaded on demand.
- It starts with the first model (`gateway_autostart`) and keeps running after models stop. It
  uses almost no resources.
- The log is at `~/.config/gatavat/logs/_gateway.log` (`gatavat logs gateway`), with the client address of each request.

## Sharing with other machines

Someone else can use your models through the gateway, for example a friend on the same Wi‑Fi.
Your models keep listening on `127.0.0.1`; only the gateway is reachable from outside.

```sh
gatavat share --enable                   # listen on all networks, create an API key
gatavat share --host 100.101.102.103     # or listen on one address only, e.g. your Tailscale IP
gatavat share --allow 192.168.1.23       # optional: only these IPs/networks (comma-separated)
gatavat share                            # print addresses, key, a curl test and an opencode provider
gatavat share --new-key                  # rotate the key (the old one stops working immediately)
gatavat share --disable                  # this machine only again (the key is kept)
```

`gatavat share` prints everything the other person needs: the address (`http://<your-mac>.local:8100/v1`,
your IP, and your Tailscale IP if Tailscale is installed), the key, a `curl` test,
`OPENAI_BASE_URL`/`OPENAI_API_KEY` for OpenAI-compatible tools, and an opencode provider block
(`llama-shared`) to merge into their `opencode.json`.

Rules the gateway applies:

- Requests from this machine never need the key, so your own opencode keeps working unchanged.
- Requests from other machines need the key and, if `gateway_allow` is set, an allowed address.
  Without a key, other machines are refused (HTTP 403), even if the gateway listens on the network.
- `/health` needs no key. Key and allow-list changes apply immediately; host changes restart the
  gateway.

Notes:

- The first time, macOS asks whether Python may accept incoming connections. Allow it, or the
  other machine times out.
- Not on the same network? Use Tailscale: see [Sharing models over Tailscale](TAILSCALE.md), a
  step-by-step guide for both the host and the guest.
- Traffic is plain HTTP. On your home network or over Tailscale (which encrypts) that's fine; don't
  expose the port to the internet.
- Their requests run on your GPU and memory. With `parallel` at 1, their requests and yours wait
  for each other; more slots let them run side by side at the cost of KV-cache memory per slot.
- Keep the web UI on `127.0.0.1`; it can start and stop processes and shows the key.

## Web UI

`gatavat ui` serves the UI on `127.0.0.1:8090` and opens the browser. Each model has a card with
status, endpoint, specs summary, Start / Restart / Stop / Logs / Chat, the context size and port,
and all other settings under "All settings & command". The header shows the gateway status,
**Start autostart models**, **Stop all**, the **opencode.json** preview and write, **Add model**,
and global **Settings**.

Stop the UI with Ctrl-C (foreground) or `gatavat ui stop` (from any terminal, also for `-d`). If it's
already running, `gatavat ui` opens it in the browser. Stopping the UI does not stop models or the gateway.

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
| devstral2-123b | Devstral 2 123B Instruct Q3_K_S (q8_0 KV cache, flash attention) | 8106 | 16384 | `~/Models/LLM/Devstral-2-123B-Q3_K_S/Q3_K_S/Devstral-2-123B-Instruct-2512-Q3_K_S.gguf` (split) |
| devstral-small2-24b | Devstral Small 2 24B Instruct Q8_0 (q8_0 KV cache, flash attention) | 8107 | 65536 | `~/Models/LLM/Devstral-Small-2-24B/Devstral-Small-2-24B-Instruct-2512-Q8_0.gguf` |

Ports follow 8100 + i; models you add get the next free port from 8108. The gateway is on 8100
and the UI on 8090.

## Updating

```sh
cd ~/Projects/gatavat && git pull   # wherever you cloned it
```

CLI commands always run the current code. Long-running gatavat processes need a restart to pick
up new code:

- **web UI:** Ctrl-C, then `gatavat ui`
- **gateway:** `gatavat gateway restart`

Running models don't need a restart; they are llama-server processes, not gatavat code. Your
config is kept. New settings appear with their defaults.

## Files

| path | contents |
|---|---|
| `~/.config/gatavat/config.json` | all settings. Set `GATAVAT_HOME` to use another folder |
| `~/.config/gatavat/logs/<id>.log` | server logs, rotated at 20 MB; `_gateway.log` for the gateway |
| `~/.config/gatavat/run/<id>.json` | pid and command of running servers |
| `~/.config/gatavat/server-flags.json` | cached `llama-server --help` flags |

## Troubleshooting

| symptom | cause / fix |
|---|---|
| "model file not found" | Wrong `model` path. `gatavat show <id>` prints the expanded path |
| "exited (see logs)" right after start | `gatavat logs <id>`. Common causes: `unknown model architecture` (llama.cpp too old, so update it), not enough memory, port in use |
| "'llama-server' not found on PATH" | `gatavat global llama_server=/full/path/to/llama-server` |
| "something is already listening on port N" | Another program uses the port. Change it with `gatavat set <id> port=...` |
| new UI features missing after an update | Restart the UI (and `gatavat gateway restart`) |
| CMake build errors after updating llama.cpp | `cmake -B build --fresh` (a stale cache can reference an old macOS SDK) |

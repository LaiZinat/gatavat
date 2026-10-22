# llmctl: run local LLMs with llama.cpp

`llmctl` is a small command-line tool and web UI for running local models with
[llama.cpp](https://github.com/ggml-org/llama.cpp)'s `llama-server`. You configure each model
once (file, port, context size, GPU offload, sampling, thinking, ...) and then start, stop and
test them from the terminal or the browser, several at the same time.

- **One config, two front ends.** The CLI and the web UI share the same settings and process
  state, so a model started in the browser can be stopped from the terminal.
- **Several models at once**, each on its own port and running in the background. Closing the UI
  or the terminal doesn't stop them.
- **Model specs without loading the model.** It reads the GGUF header for architecture,
  quantization, trained context and experts, and estimates KV-cache and total memory for the
  context you choose.
- **Built-in chat test**, streaming in the CLI and the UI, with reasoning output and tokens/s.
  For embedding models it runs an embedding and similarity test instead.
- **Gateway:** one OpenAI-compatible endpoint (`:8100/v1`) for all models, routed by model name.
- **opencode integration:** writes your models into `opencode.json` as a single provider.
- **No dependencies:** one Python file plus one HTML file, standard library only.

```text
$ llmctl
ID               STATE    PORT  CTX     PID    UP     ENDPOINT                  NOTES
qwen25-14b       stopped  8101  8192                  http://127.0.0.1:8101/v1
qwen3-coder      ready    8103  131072  41234  1h05m  http://127.0.0.1:8103/v1  autostart
qwen36-35b       loading  8104  32768   41301  0m12s  http://127.0.0.1:8104/v1
qwen3-embedding  stopped  8105  8192                  http://127.0.0.1:8105/v1

gateway running (pid 41240) -> http://127.0.0.1:8100/v1  (one endpoint for all models, routed by model name)
```

## Prerequisites

| | |
|---|---|
| **OS** | macOS (developed and tested on Apple Silicon). Linux should work but is untested. Windows is not supported. |
| **Python** | 3.8 or newer, standard library only (the macOS system Python or Homebrew's is fine) |
| **llama.cpp** | `llama-server`, recent enough for your models (newer architectures such as Qwen3.6 need recent builds) |
| **Models** | GGUF files, e.g. from [Hugging Face](https://huggingface.co/models?library=gguf) |
| **Memory** | Enough RAM/VRAM for the models you run *at the same time*. `llmctl info` estimates it. |
| optional | [opencode](https://opencode.ai) if you want the coding-agent integration |

### Install llama.cpp

```sh
brew install llama.cpp          # provides llama-server
```

Or build it from source to get the newest model support:

```sh
git clone https://github.com/ggml-org/llama.cpp ~/Projects/llama.cpp
cd ~/Projects/llama.cpp
cmake -B build                  # on Linux with NVIDIA add: -DGGML_CUDA=ON
cmake --build build --config Release -j
```

If `llama-server` is not on your PATH, tell llmctl where it is:
`llmctl global llama_server=~/Projects/llama.cpp/build/bin/llama-server`.

## Install llmctl

```sh
git clone https://github.com/LaiZinat/gatavat ~/Projects/gatavat
ln -s ~/Projects/gatavat/llmctl /opt/homebrew/bin/llmctl   # or any folder on your PATH, e.g. ~/.local/bin
llmctl --version
```

Keep `ui.html` next to the script; the symlink is followed. To update, run `git pull` and then
restart the UI and the gateway (see [Updating](docs/USAGE.md#updating)).

## Quick start

The first run creates `~/.config/llmctl/config.json` with example models: a set of Qwen models
(see [the defaults](docs/USAGE.md#built-in-example-models)). If you don't have those files,
`llmctl` marks them "model file missing". Either download them into the expected paths, or add
your own:

```sh
# 1. download a GGUF (example; any GGUF works)
hf download Qwen/Qwen2.5-7B-Instruct-GGUF --include "*q4_k_m*" --local-dir ~/Models/LLM/Qwen2.5-7B-Instruct

# 2. add it (next free port from 8106), check the specs, start it, wait until loaded
llmctl add qwen7b model=~/Models/LLM/Qwen2.5-7B-Instruct/qwen2.5-7b-instruct-q4_k_m.gguf ctx_size=32768
llmctl info qwen7b
llmctl start qwen7b -w

# 3. talk to it
llmctl chat qwen7b "Say hello in three languages"
curl http://127.0.0.1:8100/v1/chat/completions -H 'Content-Type: application/json' \
  -d '{"model": "qwen7b", "messages": [{"role": "user", "content": "hi"}]}'

# 4. or do all of the above in the browser
llmctl ui                       # http://127.0.0.1:8090
```

Remove example models you don't need with `llmctl rm <id>`.

## Documentation

- [Usage and configuration reference](docs/USAGE.md): all commands, settings, gateway, files
- [Using llmctl with opencode](docs/OPENCODE.md): provider export, an agent setup for local models, LSP
- [Changelog](CHANGELOG.md)
- [Contributing](CONTRIBUTING.md): commit message conventions

## Security

- The web UI, the gateway and the built-in models listen on `127.0.0.1` only by default.
- None of them have authentication. Only set a host to `0.0.0.0` on a network you trust,
  because anyone who can reach the port can use the model (and, for the UI, start and stop
  processes).
- The UI rejects requests from other websites (Host header check plus a required custom
  header), so a web page you visit can't control it.
- llmctl stores no API keys or credentials.
- Please report security issues privately to mail@leonhardhorstmeyer.com rather than in a public issue.

## Author

Dr. Leonhard Horstmeyer ([@leomarlo](https://github.com/leomarlo)), mail@leonhardhorstmeyer.com

## License

[MIT](LICENSE)

# Using gatavat with opencode

## Export your models

```sh
gatavat opencode            # preview the merged opencode.json
gatavat opencode --write    # write it; the existing file is backed up first (opencode.json.backup-<time>)
```

By default (`opencode_mode=gateway`) every chat model with an `opencode_provider` value goes into
one provider, `llama-local` ("llama.cpp (local)"), whose base URL is the gatavat gateway
(`http://127.0.0.1:8100/v1`). For each model:

- the key is the model's `alias`, e.g. `llama-local/qwen3-coder-next`
- `limit.context` comes from `ctx_size`, and `limit.output` from `opencode_output`

Embedding models are never exported. Everything else in the file (other providers, agents,
`lsp`, ...) is kept. If models were previously in separate per-model providers, they are moved,
and the top-level `model` / `small_model` and agents' `model` references are rewritten.

Run `--write` again whenever you change `ctx_size`, `opencode_output`, ports or the set of
models, then restart opencode. It reads its config only at startup.

The models must be running for opencode to use them: `gatavat start <id>` also starts the gateway.

Options:

- `gatavat global opencode_mode=per-model`: one provider per model, each pointing at the model's
  own port (no gateway needed).
- `gatavat global opencode_path=~/.config/opencode/profiles/x.json`: write into another config
  file, e.g. one you load with `OPENCODE_CONFIG=...`.
- `gatavat set <id> opencode_provider=`: leave a model out of opencode.

## Start opencode with a profile

```sh
gatavat oc laizinat                     # = OPENCODE_CONFIG=~/.config/opencode/profiles/laizinat.json opencode
gatavat oc laizinat -s qwen3-coder      # start the model first (if needed) and wait until it's ready
gatavat oc laizinat -- ~/Projects/app   # everything after -- goes to opencode
gatavat oc ~/somewhere/else.json        # a path works too
gatavat oc                              # opencode_default_profile, or opencode's normal config if blank
gatavat oc default                      # opencode's normal config, ignoring the default profile
gatavat oc --list                       # available profiles
```

`gatavat code` is an alias. Profiles are `*.json` files in `opencode_profiles_dir`. opencode
merges the profile with its global config, so providers written by `gatavat opencode --write` stay
available. Set a default with `gatavat global opencode_default_profile=laizinat`.

## An agent setup for local models

Local models do better with a narrower scope. This agent (add it under `"agent"` in
`opencode.json`) forbids subagents and web fetches, asks before edits and unknown shell
commands, and pre-approves read-only git commands and build/test commands:

```json
"agent": {
  "local-careful": {
    "description": "Controlled local coding agent",
    "mode": "primary",
    "model": "llama-local/qwen3-coder-next",
    "temperature": 0.7,
    "top_p": 0.8,
    "steps": 30,
    "permission": {
      "task": "deny",
      "edit": "ask",
      "webfetch": "deny",
      "doom_loop": "deny",
      "bash": {
        "*": "ask",
        "git status*": "allow", "git diff*": "allow", "git log*": "allow",
        "cargo check*": "allow", "cargo build*": "allow", "cargo test*": "allow",
        "cargo clippy*": "allow", "cargo fmt*": "allow",
        "forge build*": "allow", "forge test*": "allow", "forge fmt*": "allow",
        "npm test*": "allow", "pnpm test*": "allow"
      }
    }
  }
}
```

Notes:

- **Don't use temperature 0.** Greedy decoding makes repetition loops more likely; Qwen
  recommends against it. Use the model vendor's recommended sampling.
- **Don't set `steps` too low.** Reading, searching, editing and testing quickly add up to 10+
  steps for a small task.
- **Keep opencode's default system prompt** (no `prompt` key). Put general rules in `AGENTS.md`
  instead. Example:

  ```markdown
  - Read the relevant code before editing it.
  - Work in small, verified steps: change one file, then build, lint or test it.
  - After an error, read it carefully; never repeat an identical failing call.
  - After two failed attempts at the same problem, stop and explain the blocker.
  ```

## Context size

opencode's system prompt and tool definitions take a sizeable part of the context, so 8k is too
small and 32k is tight. Check what a larger context costs with `gatavat info <id>`: the KV-cache
estimate scales linearly with `ctx_size`. Hybrid models (Qwen3-Next, Qwen3.5/3.6) need far less
KV memory per token than classic transformers, so for them 128k is usually cheap. For example:

```sh
gatavat set qwen3-coder ctx_size=131072
gatavat opencode --write && gatavat restart qwen3-coder
```

opencode compacts (summarizes) the conversation as it approaches the limit you advertise.

## LSP

opencode's LSP support is off unless the config has an `lsp` key (see
[opencode's LSP docs](https://opencode.ai/docs/lsp/)). An object enables the built-in servers
and lets you add your own, for example Solidity:

```json
"lsp": {
  "solidity": {
    "command": ["nomicfoundation-solidity-language-server", "--stdio"],
    "extensions": [".sol"]
  }
}
```

The language server programs must be installed (e.g. `rustup component add rust-analyzer`,
`npm i -g @nomicfoundation/solidity-language-server`). Servers start when the agent first reads
a matching file. `gatavat opencode --write` keeps the `lsp` block.

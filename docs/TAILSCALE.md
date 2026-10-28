# Sharing models over Tailscale

This guide is for two people:

- the **host**, whose computer runs the models with gatavat, and
- the **guest**, for example a family member, who wants to use those models from their own
  computer, at home or anywhere else.

[Tailscale](https://tailscale.com) builds a private, encrypted network between your devices over
the internet. The host shares one computer with the guest. The guest can then reach it at a fixed
address, as if both were on the same Wi‑Fi. Nothing has to be opened on the router, and the rest of
the internet can't see the models.

```text
guest's laptop ──(Tailscale, encrypted)──> host's Mac :8100  gatavat gateway ──> llama-server models
   opencode / OpenHands / curl               API key required        (127.0.0.1 only)
```

The guest doesn't need gatavat or llama.cpp, only Tailscale and the program they want to use the
models with.

- [Part 1: Host](#part-1-host)
- [Part 2: Guest](#part-2-guest)
- [Troubleshooting](#troubleshooting)
- [Why not SSH, port forwarding or HTTPS?](#why-not-ssh-port-forwarding-or-https)

## Part 1: Host

### 1. Install Tailscale and sign in

```sh
brew install --cask tailscale-app
```

Or get it from the [Mac App Store](https://apps.apple.com/app/tailscale/id1475387142) or
[tailscale.com/download](https://tailscale.com/download). Open Tailscale from Applications, allow
the VPN configuration and the network extension when macOS asks (System Settings → General →
Login Items & Extensions → Network Extensions → Tailscale on), and sign in. macOS warns that these
"can monitor network traffic"; it says that for every VPN app. Tailscale only carries traffic to
`100.x` addresses unless you turn on an exit node. The free Personal plan is enough. Your Mac now
has a Tailscale address starting with `100.`, shown in the menu bar app.

Under Tailscale's **Settings**, turn on **Launch at login** so the address keeps working after a
restart.

### 2. Turn on sharing in gatavat

```sh
gatavat share --enable --allow 100.64.0.0/10
```

- `--enable` makes the gateway listen on the network and creates an API key.
- `--allow 100.64.0.0/10` accepts only Tailscale addresses (all of them start with `100.64`
  to `100.127`), so other devices on your Wi‑Fi are refused even if they somehow get the key.
- To also share with someone on your home Wi‑Fi, add their network, for example
  `--allow "100.64.0.0/10, 192.168.1.0/24"`.

The first time another computer connects, macOS asks whether Python may accept incoming
connections. Click **Allow**.

> Alternative: `gatavat share --host <your 100.x address>` listens on the Tailscale address only.
> That gateway can't start until Tailscale is connected, though, so after a restart you may have
> to run `gatavat gateway restart`. `--enable --allow ...` doesn't have that problem.

### 3. Start the models they should use

```sh
gatavat start qwen36-35b -w
```

Only running models answer. The guest gets HTTP 503 "model … is not running" for the others.

### 4. Share your Mac with the guest

1. Open the [Tailscale admin console](https://login.tailscale.com/admin/machines) and go to
   **Machines**.
2. Click the **⋯** menu next to your Mac and choose **Share…**
3. Either enter the guest's email address (**Share by email**), or copy an invite link
   (**Copy invite link**) and send it to them. Treat the link like a password. It expires after
   30 days.

Sharing gives the guest your Mac only, not your other devices. Sharing is available on all
Tailscale plans; check Tailscale's pricing page for whether recipients count towards your plan.

### 5. Limit guests to the gatavat port

A new tailnet's access policy allows everything, so a guest could reach **every** service on your
Mac that listens on all addresses: AirPlay Receiver, dev servers, file or screen sharing. Check
what that includes:

```sh
lsof -nP -iTCP -sTCP:LISTEN | grep -v -e 127.0.0.1 -e '\[::1\]'
```

In the admin console, open **Access controls** and replace the `grants` (or `acls`) section with
the following. Keep any `ssh` section as it is.

```json
"grants": [
  { "src": ["autogroup:member"], "dst": ["*"], "ip": ["*"] },
  { "src": ["autogroup:shared"], "dst": ["*"], "ip": ["tcp:8100"] }
]
```

The first rule keeps full access between your own devices. The second lets people you shared a
machine with reach port 8100 only. They only ever see the machines you shared, so `"dst": ["*"]`
means just those. Ask the guest to check: `/v1/models` on port 8100 answers, any other port
times out.

### 6. Send the guest their settings

```sh
gatavat share
```

This prints the addresses (look for the lines marked **Tailscale**), the API key, a test command,
and ready-made settings for opencode and other tools. Send the guest:

- the address `http://<your-mac>.<tailnet>.ts.net:8100/v1`, or the `100.x` address,
- the API key (send it privately, e.g. in a messenger, not in a public chat), and
- the opencode block, if they use opencode.

### Later

| task | command |
|---|---|
| check what's shared and who could connect | `gatavat share` |
| see who is using it | `gatavat logs gateway -f` (client address per request) |
| after a restart of your Mac | `gatavat start <models>`. The gateway starts with the first model |

### Stopping and locking out

From mildest to strongest:

| goal | how | undo |
|---|---|---|
| lock everyone out of gatavat at once, e.g. a key leaked | `gatavat share --new-key`. The old key stops working immediately; send the new one (`gatavat share`) only to those who keep access | (keys can't be restored) |
| remove one person for good | admin console → **Machines** → your Mac → ⋯ → **Share…** → remove them. Also run `gatavat share --new-key` if they had the key | share again |
| stop sharing the models, keep using them yourself | `gatavat share --disable`. The gateway listens on `127.0.0.1` only again; the key is kept | `gatavat share --enable` |
| stop the gateway entirely | `gatavat gateway stop`. Your own opencode loses its endpoint too | `gatavat gateway start` |
| take your Mac off Tailscale for now | menu bar icon → **Disconnect**, or `tailscale down` | **Connect**, or `tailscale up` |

The `tailscale` command is inside the app:
`alias tailscale=/Applications/Tailscale.app/Contents/MacOS/Tailscale` (add it to `~/.zshrc` to
keep it).

To check that nothing else is reachable, list what listens beyond this machine:

```sh
lsof -nP -iTCP -sTCP:LISTEN | grep -v -e 127.0.0.1 -e '\[::1\]'
```

While sharing, expect `Python … *:8100` (the gateway). `rapportd` is Apple's Continuity and
`ControlCe … *:5000, *:7000` is AirPlay Receiver (System Settings → General → AirDrop & Handoff),
both fine to keep. Anything else, e.g. a dev server, should be stopped or bound to `127.0.0.1`;
the access rule from step 5 blocks it for guests either way.

Their requests run on your computer. Big models slow down when two people use them at once, and
with `parallel` at 1 each request waits for the previous one.

## Part 2: Guest

### 1. Install Tailscale and accept the invite

- **Mac:** `brew install --cask tailscale-app`, or the Mac App Store, or
  [tailscale.com/download](https://tailscale.com/download)
- **Windows / Linux:** [tailscale.com/download](https://tailscale.com/download)

Create a free Tailscale account (any Google, Microsoft, Apple or GitHub login) and sign in. Then
open the invite link the host sent you and accept it. The host's computer now appears in your
Tailscale app under your machines, with a `100.x` address.

### 2. Check the connection

Replace the address and key with the ones the host gave you:

```sh
curl http://<host-mac>.<tailnet>.ts.net:8100/v1/models -H "Authorization: Bearer llm-…"
```

You should get a list of models with their `state`. Then a real question:

```sh
curl http://<host-mac>.<tailnet>.ts.net:8100/v1/chat/completions \
  -H "Authorization: Bearer llm-…" -H "Content-Type: application/json" \
  -d '{"model": "qwen3.6-35b-a3b", "messages": [{"role": "user", "content": "Hello!"}]}'
```

If the name doesn't resolve, use the `100.x` address instead. The `.ts.net` name requires
**MagicDNS**, which is on by default in new Tailscale accounts (admin console → DNS).

### 3. Use it in your program

The host's computer behaves like an OpenAI-compatible API. Every program needs three values:

| setting | value |
|---|---|
| Base URL | `http://<host-mac>.<tailnet>.ts.net:8100/v1` |
| API key | the `llm-…` key from the host |
| Model | a model name from `/v1/models`, e.g. `qwen3.6-35b-a3b` |

**opencode.** Add the block that `gatavat share` printed to `~/.config/opencode/opencode.json`
(merge it into an existing `"provider"` section), restart opencode and choose
`llama-shared/<model>` with `/models`. It looks like this:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "provider": {
    "llama-shared": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "llama.cpp (host)",
      "options": {
        "baseURL": "http://<host-mac>.<tailnet>.ts.net:8100/v1",
        "apiKey": "llm-…",
        "timeout": 600000
      },
      "models": {
        "qwen3.6-35b-a3b": { "name": "Qwen3.6 35B A3B", "limit": { "context": 32768, "output": 8192 } }
      }
    }
  }
}
```

Keep the `limit.context` values the host gave you. If you set them higher than the host's
context size, long conversations fail.

**OpenHands.** Settings → LLM → Advanced: Custom model `openai/qwen3.6-35b-a3b`, Base URL as
above, API key as above. OpenHands runs in Docker; use the `100.x` address there, because the
`.ts.net` name may not resolve inside the container.

**Scripts and other tools** (OpenAI SDKs, Continue, Open WebUI, …):

```sh
export OPENAI_BASE_URL=http://<host-mac>.<tailnet>.ts.net:8100/v1
export OPENAI_API_KEY=llm-…
```

### If you run gatavat yourself

You can use your own local models and the host's at the same time: your models through your
`llama-local` provider, theirs through `llama-shared`. Both providers live side by side in
`opencode.json`, and `gatavat opencode --write` leaves `llama-shared` untouched.

## Troubleshooting

| what you see | cause and fix |
|---|---|
| `curl` hangs, then times out | The host's Mac is asleep or off, Tailscale is disconnected on either side, or macOS blocked Python's incoming connections (host: System Settings → Network → Firewall → Options, allow Python). Check with `tailscale ping <host-mac>` |
| `Connection refused` | The gateway isn't running or doesn't listen on the network. Host: `gatavat share` and `gatavat gateway restart` |
| `401 missing or wrong API key` | The key is missing, mistyped or was rotated. Ask the host for the current one |
| `403 … not allowed to use this gateway` | Your address isn't in the host's `gateway_allow`. Host: check `gatavat share` |
| `403 this gateway has no API key` | Host: `gatavat share --new-key` |
| `503 model … is not running` | Host: `gatavat start <id>` |
| name doesn't resolve | Use the `100.x` address, or turn on MagicDNS in your Tailscale admin console |
| slow answers | The host's computer is busy (their own use, or someone else's). Smaller models answer faster |

The host's Mac must be awake. For a Mac that serves others regularly, turn on System Settings →
Energy → **Prevent automatic sleeping when the display is off** (on a laptop: only while plugged
in).

## Why not SSH, port forwarding or HTTPS?

- **Router port forwarding** puts the models on the public internet. The key protects them, but the
  traffic is unencrypted HTTP and the port gets scanned constantly. Tailscale needs no open ports.
- **SSH tunnels** work, but the host has to turn on Remote Login and give the guest an SSH key to
  their computer, which must then be restricted to port forwarding. Tailscale sharing is simpler
  and only exposes the one machine.
- **HTTPS isn't needed.** Tailscale already encrypts everything between the two devices, so plain
  `http://` over Tailscale is private. `tailscale serve` can add an HTTPS address, but then requests
  reach the gateway from the host's own machine. gatavat recognizes such relayed requests by their
  proxy headers and still requires the key, but it can't see the guest's address, so with
  `gateway_allow` set they are refused. Prefer the plain setup above.

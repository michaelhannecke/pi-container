<p align="center">
  <img src="blog-image.jpg" alt="pi coding agent in an Apple Container" width="100%">
</p>

<h1 align="center">pi-container</h1>

<p align="center">
  <strong>A sovereign, npm-free local coding agent on macOS.</strong><br>
  The <code>pi</code> coding agent runs in a disposable Apple <code>container</code> micro-VM and talks to a local
  MLX-Swift model on the host — no Node, no npm, no agent binary on your work machine.
</p>

<p align="center">
  <img src="https://img.shields.io/badge/platform-macOS%2026%20(Tahoe)%20%C2%B7%20Apple%20Silicon-black" alt="platform">
  <img src="https://img.shields.io/badge/runtime-Apple%20container-blue" alt="runtime">
  <img src="https://img.shields.io/badge/agent-pi--coding--agent%20%C2%B7%20Node%2022-339933" alt="agent">
  <img src="https://img.shields.io/badge/model-gemma--4--26b%20%C2%B7%20MLX--Swift-orange" alt="model">
  <img src="https://img.shields.io/badge/status-hands--on%20draft-yellow" alt="status">
  <img src="https://img.shields.io/badge/license-MIT-green" alt="license">
</p>

---

## Overview

A modern coding agent reads your files, runs shell commands, and installs whatever it decides it needs. On a work machine in a regulated context that is an unacceptable blast radius. This repository contains a **runnable setup** that closes it:

- **Inference stays native on the host** — MLX-Swift needs Apple Silicon's Metal/ANE, which a Linux VM does not expose.
- **The agent runtime is sandboxed in its own VM** — Apple `container` gives each container a lightweight VM, not shared-kernel namespaces.
- **The host stays clean** — no Node, no npm, no `pi` binary; the agent lives only inside an image and is discarded on exit.

The full step-by-step walkthrough is the article **[`en-pi-apple-container.md`](en-pi-apple-container.md)** (English companion to a German MLX-Swift writing series). The files in this repo are the runnable reference for that article — change one, change the other.

## Architecture

```
┌─────────────────────────────┐        ┌──────────────────────────────┐
│ Host (macOS, Apple Silicon) │        │ Apple Container (Linux VM)   │
│                             │        │                              │
│  MLX-Swift server           │◄──────►│  pi-coding-agent             │
│  /v1/chat/completions       │ Bridge │  (Node 22, ripgrep, git)     │
│  gemma-4-26b-4bit           │        │  Workspace: /workspace       │
└─────────────────────────────┘        └──────────────────────────────┘
```

- **Inference** runs on the host (it has to — no Metal/ANE in a Linux VM).
- **Tool-calling sandbox** runs in the container — a clean split between model runtime and agent runtime.
- **pi** reaches the host only over the container bridge; the gateway IP is environment-dependent and discovered at runtime, never hardcoded.

## Table of contents

- [Repository structure](#repository-structure)
- [Prerequisites](#prerequisites)
- [Quickstart](#quickstart)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)
- [The article](#the-article)
- [Notes & caveats](#notes--caveats)
- [License](#license)

## Repository structure

```
.
├── en-pi-apple-container.md                      # the step-by-step article (English)
├── blog-image.jpg                                # article / README banner
├── Containerfile                                 # node:22-bookworm-slim + pi installed globally
├── pi-config/
│   ├── AGENTS.md                                 # global agent rules (container variant)
│   ├── models.json                               # provider + model definition
│   └── extensions/
│       └── protected-paths/
│           └── index.ts                          # tool-call guardrail for sensitive paths
└── scripts/
    ├── build.sh                                  # container build
    └── run.sh                                    # container run with the right mounts
```

`pi-config/` is mounted into the container at runtime as the agent's config directory. Its `sessions/`, `cache/`, and `logs/` subdirectories are produced by pi during a session and are git-ignored — runtime artifacts, not configuration.

## Prerequisites

- **macOS 26 (Tahoe) on Apple Silicon, recommended.** `container` technically runs on macOS 15, but its networking is significantly limited there and this whole setup lives or dies on container-to-host networking. Treat macOS 15 as unsupported here.
- Apple `container` CLI installed (`container --version` must answer).
- **macOS Local Network permission grantable** — recent macOS gates local traffic behind a privacy prompt; it must be allowed for the container runtime.
- A local model server running **on the host** with an OpenAI-compatible `/v1/chat/completions` endpoint, serving the model you've loaded (e.g. `gemma-4-26b-a4b-it-4bit`), bound to `0.0.0.0:8080` (not only `127.0.0.1`). Native tool-calling is model-dependent — verify it for an agent workflow.
- **No Node and no npm on the host** — that is the point; the agent lives only in the image.

## Quickstart

### 1. Build the image

```bash
./scripts/build.sh
```

Produces `pi-coding-agent:local` (override the tag with `IMAGE_TAG=...`).

### 2. Discover the host bridge IP

From inside the container, the host is reachable via the bridge's default gateway. The address is environment-dependent, so discover it instead of assuming a subnet. The image's entrypoint is `pi`, so override it with `--entrypoint sh` for a one-off command (otherwise `pi` starts and reports "No API key found for the selected model"):

```bash
container run --rm --entrypoint sh pi-coding-agent:local -c "ip route | awk '/default/ {print \$3}'"
```

If the printed gateway differs from the default in `pi-config/models.json`, update `providers.mlx-local.baseUrl` accordingly (keep the `:8080/v1` suffix).

### 3. Run the agent

```bash
PROJECT_DIR=~/projects/your-repo ./scripts/run.sh --model mlx-local/gemma4-instruct
```

`run.sh` mounts exactly two things, and nothing else crosses the boundary:

- `pi-config/` → `/home/pi/.pi/agent` (provider config, `AGENTS.md`, extensions)
- `$PROJECT_DIR` → `/workspace` (the project being worked on)

`--rm` discards the VM and its writable layer on exit. The host is byte-for-byte unchanged.

## Configuration

### Models & provider — `pi-config/models.json`

Defines the `mlx-local` provider with `api: "openai-completions"` and a nested `models` array. `apiKey` is `"not-required"` (a local server needs no secret, so none can leak). Each model's `id` must be **exactly** what the server's `/v1/models` reports — here the full model path `/models/gemma-4-26b-a4b-it-4bit` (replace with the absolute path your host server actually returns) — while `name` (`mlx-local/gemma4-instruct`) is the handle `--model` matches against. `contextWindow`/`maxTokens` are optional; omitted here, so pi's defaults (128K / 16.4K) apply — set them to your server's real limits.

### Global agent rules — `pi-config/AGENTS.md`

Loaded into every session as the operating contract: runs in an Apple container, host not directly reachable, file operations only affect `/workspace`, model reached only over the bridge, no external calls or telemetry without explicit instruction, and tool discipline (`read` before `edit`, `write` only for new files, no `npm install` without confirmation).

### Extension — `protected-paths`

A defense-in-depth backstop at `pi-config/extensions/protected-paths/index.ts`. It hooks pi's `tool_call` event and forces a confirmation (or hard-denies) for sensitive directories (`~/.ssh`, `~/.aws`, `~/.config/gcloud`, `/run/secrets`, `/etc`) and patterns (`.env`, `credentials.json`, `id_rsa`, `id_ed25519`, `*.pem`, `*.p12`) — inspecting both file-tool paths and `bash` commands. The container is the strong boundary; this is the seatbelt for the day someone widens a mount.

## Troubleshooting

| Symptom | Cause & fix |
|---|---|
| Requests hang/fail with no error, empty reply | **Local Network permission not granted.** *System Settings → Privacy & Security → Local Network* — enable the container runtime, then fully quit and reopen the requesting app. Most common first-run failure on recent macOS. |
| "Can't reach the model" | **Host bound to loopback.** The container is a separate VM and cannot reach host `127.0.0.1`. Bind the model server to `0.0.0.0:8080`. |
| Connection refused / wrong address | **Wrong bridge IP.** `192.168.64.1` is only a default — re-run the `ip route` discovery and use the actual gateway. |
| Files not owned by your macOS user | **Expected.** The container writes as UID 1000; your host user is typically UID 501. In the pi workflow (edits go through the `edit` tool) this is acceptable. |
| Agent answers but never edits | **No native tool-calling.** pi has no `toolCalling` flag — it relies on the model doing OpenAI function-calling. Some instruct builds (Gemma included) may not, and silently no-op. Verify with a real session. |
| `models.json` loads but chat fails on role/params | Some local servers reject the `developer` role or `reasoning_effort`. Add provider-level `"compat": { "supportsDeveloperRole": false, "supportsReasoningEffort": false }` (see pi's `models.md`). |

## The article

**[`en-pi-apple-container.md`](en-pi-apple-container.md)** is a hands-on draft for external publication. It covers, in order: why a coding agent belongs in a container, why Apple `container` over Docker on Apple Silicon, then a careful nine-step walkthrough (install `container` → verify the host model → build → discover the bridge IP → wire `models.json` → guardrails → run → smoke-test) plus troubleshooting.

The code blocks in the article mirror the `Containerfile`, `pi-config/`, and `scripts/` files in this repo verbatim. **Keep them consistent** — if you change a runnable file, update the article, and vice versa.

## Notes & caveats

- **MLX stays on the host.** Suggestions to move inference into the container do not work — no Metal, no ANE in a Linux VM.
- **Bridge IP is environment-dependent.** The address in `models.json` is an example default, discovered at runtime; it can vary by `container` version.
- **macOS version matters.** Container-to-host networking is the linchpin; older macOS limits it severely.

## License

Licensed under the MIT License — see [`LICENSE`](LICENSE).

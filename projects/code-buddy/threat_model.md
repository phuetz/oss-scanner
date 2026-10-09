# Threat model — Code Buddy

## What this project does and where untrusted input enters
Code Buddy (`buddy`) is a terminal AI coding agent. A language model plans and calls tools that read and write files, run shell commands, browse the web and call MCP servers. It can also run as a server (`buddy server`: HTTP API, WebSocket `/ws`, mobile PWA) and as a fleet of peers that invoke each other's models and read-only tools.

Untrusted input:
- **repository content** in the working directory: `.codebuddy/` (hooks.json, settings, MCP config, skills), `CODEBUDDY.md`/`AGENTS.md` context files, plugin manifests. A user may open a malicious repository: until the folder is trusted, nothing in it may execute or obtain secrets;
- **model output**, which may be prompt-injected through web pages, files or tool results: tool calls must stay within the confirmation, permission-mode and sandbox rules;
- **network input** to `buddy server` (HTTP, WebSocket, channel adapters such as Telegram) and to fleet peers (`peer.tool.invoke`);
- skills and skill packages imported from external libraries (the skill firewall is in scope).

## Components that matter most / least
Most important:
- folder trust and hooks (`src/hooks/`, `src/security/trust-folders.ts`): nothing from an untrusted repository may run, and hooks must never receive the full environment with API keys;
- shell execution (`src/tools/bash*`, `src/security/`): command validation, environment filtering (secrets), the optional native sandbox (`CODEBUDDY_NATIVE_SANDBOX`), permission modes and the confirmation service;
- file tools: path traversal outside the workspace, symlink following;
- the server and fleet (`src/server/`, `src/fleet/`): authentication (JWT), origin checks, the `peer.tool.invoke` gates (allowlist, `fleetSafe`, workspace root, fail-closed);
- secret handling: keys in config files, logs, transcripts, session files, and their permissions.

Less important: `cowork/` (desktop GUI, a separate package), demos, docs, the Rust sidecars (`buddy-sense/`, `buddy-memory/`) unless reachable from the network.

## How to exercise it
`buddy --help`; `buddy -p "<prompt>"` runs a headless turn (needs a model provider: an OpenAI-compatible endpoint can be faked locally). `buddy server --port 3000` starts the server. Security regression tests are in `tests/security/` and `tests/hooks/`. Use an isolated HOME and fake API keys (for example `OPENAI_API_KEY=sk-test-not-real`).

## How you rate severity
- Code execution or secret disclosure triggered only by opening or working in an untrusted repository: critical.
- Bypass of the confirmation/permission mode or of the sandbox by model output (prompt injection) leading to file writes or command execution: high to critical.
- Unauthenticated remote access to the server, or a fleet peer reading files outside the configured workspace: critical.
- Secrets written to world-readable files or logs: high.
- Denial of service of the local CLI: low.

## Anything to leave alone
`bypassPermissions`/YOLO modes are explicit user opt-ins that disable confirmations by design; report only what escapes their documented guardrails. Features gated behind opt-in environment variables that are off by default are in scope only when you state the configuration used.

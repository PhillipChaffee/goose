# Connect-failure forensics: Goose Mobile vs current goose server (v1.51.x)

Ticket: [PhillipChaffee/goose#3](https://github.com/PhillipChaffee/goose/issues/3) (wayfinder map #1).
Sources: shallow clone of [aaif-goose/goose-mobile](https://github.com/aaif-goose/goose-mobile) at `main` (2026-09-19), plus cheap re-verification of server facts against this repo (`crates/goose`).

## Verdict

The Goose Mobile iOS app does **not** speak ACP at all. It speaks the **pre-v1.42 `goosed` REST API** — `POST /reply` with SSE streaming, `/agent/*`, `/config/*`, `/sessions/*` — authenticated with `X-Secret-Key`. That REST server (`crates/goose-server`) was deleted on 2026-07-07 (shipped v1.42.0), so every endpoint the app calls returns 404 on a current `goose serve`, which only serves ACP V1 JSON-RPC at `/acp` (via the official `agent-client-protocol-http` crate, v1.40.0+).

Fix classification: **(c) transport rewrite on the client** — not (a) dependency bump, not (b) method migration. There is no overlap between the app's REST surface and the server's `/acp` JSON-RPC surface: different endpoints, different request bodies, different streaming shape (SSE events vs JSON-RPC/ACP session updates). A secondary script-level fix is needed for the bundled-server/tunnel path (see below).

## What the app actually speaks (evidence, goose-ios)

All connection logic lives in `goose-ios/Goose/GooseAPIService.swift` (943 lines). Endpoint inventory, every one with `X-Secret-Key` header auth (`GooseAPIService.swift:153` and 9 further call sites):

| Endpoint | Method | Where |
|---|---|---|
| `/reply` (SSE chat stream) | POST | `GooseAPIService.swift:127` |
| `/agent/start` | POST | `:256` |
| `/agent/resume` | POST | `:326` |
| `/agent/update_from_session` | POST | `:367` |
| `/agent/update_provider` | POST | `:482` |
| `/agent/add_extension` | POST | `:585` |
| `/config/read` | GET | `:428` |
| `/config/extensions` | GET | `:535` |
| `/sessions` | GET | `:678` |
| `/sessions/{id}` | GET | `:299` |
| `/sessions/insights` | GET | `:631` |

- Request body for chat is the old goosed shape: `ChatRequest { user_message, override_conversation, session_id, recipe_name, recipe_version }` — `goose-ios/Goose/Message.swift:367-381`.
- Streaming is SSE with goosed event types `Message`, `Error`, `Finish`, `ModelChange`, `Notification`, `UpdateConversation`, `Ping` — `Message.swift:384-420`, SSE delegate in `GooseAPIService.swift:728+`.
- **No WebSocket client, no JSON-RPC, no `/acp` anywhere in the repo.** All traffic is `URLSession` data tasks (`GooseAPIService.swift:219-685`).
- The upstream README confirms intent, not implementation: *"We plan to migrate the client to use a http/remote version of the ACP"* — `README.md` (roadmap section).
- Defaults match the old bundled server: base URL `http://127.0.0.1:62996`, secret `test` (`GooseAPIService.swift:12,16`); trial mode points at `demo-goosed.fly.dev` (`:21`).
- The `goosechat://configure?data=…` URL scheme (`ConfigurationHandler.swift:267`) delivers `{base_url, secret}` from the host-side launch script — this part is protocol-neutral and can stay.

## What the current server speaks (verified, cheap)

In this repo: `crates/goose-server` no longer exists (deleted v1.42.0). The server surface is ACP V1 JSON-RPC over HTTP POST + WebSocket at `/acp` — `crates/goose/src/acp/transport/mod.rs:103` (`path: "/acp"`) — built on the official `agent-client-protocol-http` crate (`crates/goose/src/acp/server.rs:2677`). Auth unchanged and compatible: `X-Secret-Key` header (`crates/goose/src/acp/transport/auth.rs:22`) or `?token=` query (`crates/goose/tests/acp_transport_auth_test.rs:192`). So auth would already work; the protocol would not.

## Secondary failure mode: the launch/tunnel path is also fossilized

The app reaches a desktop agent via its own scripts, not `goose roam` pairing (zero matches for roam/iroh in the repo):

- `launch_goosed.sh:7` downloads a **third-party prebuilt `goosed` binary** from `github.com/michaelneale/goose-tunnel/releases/download/test/goosed`, starts it with the removed `goosed agent` subcommand (`:114`), and health-checks the removed `/health` REST route (`:120`). On a machine with current goose, no such binary/subcommand exists.
- `launch_tunnel.sh` and `launch_tailscale.sh` do the same, plus a Cloudflare Worker tunnel (`michaelneale/lapstone-tunnel`, worker `cloudflare-tunnel-proxy.michael-neale.workers.dev`). The tunnel itself is a transport-neutral HTTP reverse proxy — it would still work in front of `goose serve` — but the server binary it wraps is the pre-v1.42 goosed.
- Client-side tunnel UX (503 → "tunnel disabled", Tailscale login detection, `AppNoticeCenter` notices) is address-agnostic and survives a protocol migration.

So even a user who somehow had an old REST server reachable would be on a stale third-party build; the supported path (`goose serve`) cannot be reached by this app today.

## Fix classes and rough size (in the observed goose-mobile codebase)

| Class | Applicable? | Size |
|---|---|---|
| (a) dependency bump | No — the client has no ACP dependency to bump; nothing to bump fixes a protocol the app never spoke | — |
| (b) method migration/renames | No — none of `_goose/*` or ACP method churn matters; the app predates ACP usage entirely | — |
| (c) transport rewrite | **Yes — primary fix** | ~1–2k lines Swift: rewrite `GooseAPIService.swift` (~700 of 943 lines) to JSON-RPC POST + SSE/WS against `/acp` (session/new, session/prompt, session/update streaming via `agent-client-protocol` types), replace the SSE event model in `Message.swift` (~130 lines) with ACP session-update decoding, and rewire ChatView callbacks (stream shape stays callback-like, so view-layer churn is moderate). Auth code carries over unchanged. |
| (d) different cause | Partial — not the root cause of the protocol failure, but the bundled-server path needs repair regardless: point `launch_goosed.sh`/`launch_tunnel.sh`/`launch_tailscale.sh` at current `goose serve` (GOOSE_PORT/GOOSE_SERVER__SECRET_KEY) instead of the archived `goosed agent` binary, and health-check `/acp` instead of `/health`. Small (~30–60 lines of script edits). `goose roam` pairing is *not* required by this app. |

No server-side change is needed: `goose serve` already exposes exactly the auth (`X-Secret-Key` / `?token=`) the app already knows how to send, on a route (`/acp`) the app must learn to speak.

## References

- goose-mobile @ main: `goose-ios/Goose/GooseAPIService.swift`, `goose-ios/Goose/Message.swift`, `goose-ios/Goose/ConfigurationHandler.swift`, `goose-ios/launch_goosed.sh`, `goose-ios/launch_tunnel.sh`, `goose-ios/launch_tailscale.sh`, `README.md` (roadmap).
- goose (this repo): `crates/goose/src/acp/transport/mod.rs`, `crates/goose/src/acp/transport/auth.rs`, `crates/goose/src/acp/server.rs`, `crates/goose/tests/acp_transport_auth_test.rs`; absence of `crates/goose-server`.
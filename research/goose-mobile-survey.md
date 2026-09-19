# Goose Mobile survey — artifacts, stacks, transports, and the base to update

Research ticket: [PhillipChaffee/goose#2](https://github.com/PhillipChaffee/goose/issues/2) (part of map #1).
Subject: `aaif-goose/goose-mobile` — surveyed 2026-09-19 via the GitHub API, raw source fetches, PR/issue data, the App Store listing, and the local `goose` checkout (`PhillipChaffee/goose` @ workspace version 1.51.0). Sources are cited per claim.

## 1. Repo shape and health

| Fact | Value | Source |
|---|---|---|
| Repo | `aaif-goose/goose-mobile`, public, **not archived**, not a fork | `gh api repos/aaif-goose/goose-mobile` |
| Created | 2025-03-04 (originally described "website"; consolidated into the mobile repo by PR #156 "chore: migrate mobile apps", 2026-01-19) | repo metadata; `commits?per_page=30` |
| Default branch | `main` | repo metadata |
| License | Apache-2.0 (added to repo root 2026-06-30, PR #169); per-artifact LICENSE files in both subprojects | repo metadata; tree listing |
| Language (GitHub) | Swift (goose-ios dominates the tree) | repo metadata |
| Stars / forks | 163 / 34 | repo metadata |
| Last push to `main` | 2026-06-30 (license file only; last *code* commit 2026-03-16) | `commits?per_page=30` |
| Last push anywhere | 2026-08-20 — the `mobile-web-client` branch (PR #171); `roam-web` branch 2026-08-11 | branch list with tip dates |
| Open issues / PRs | 17 open issues (mostly Android-agent feature requests); 2 open PRs: #171 (mobile-web client), #172 (third-party pairing README note) | issue/PR lists |
| Docs homepage in metadata | `https://block.github.io/goose/docs/experimental/goose-mobile/` → **HTTP 404** (stale metadata; page retired or moved) | webfetch |
| Live deployment | GitHub Pages serves `https://aaif-goose.github.io/goose-mobile/` from the `roam-web` branch (confirmed via `pages` API) | `gh api .../pages` |

Verdict: **dormant-ish on `main`, alive on the web-client front.** The iOS and Android artifacts last saw code changes 2026-03-16; the only active development in the repo since then is the browser roam client (PR #171, updated 2026-08-20) and its Pages deployment (roam-web branch, 2026-08-11). The repo was consolidated from `block/goose`-adjacent sources (original iOS client: `michaelneale/goose-ios` per README).

## 2. Artifact inventory

### 2.1 `goose-ios/` — native iOS remote client (shipped)

- **Stack**: Swift/SwiftUI Xcode project (`Goose.xcodeproj`, `Goose/*.swift` — ~36 view/manager files: ChatView, voice input/output managers, sessions, trial mode). [tree `main:goose-ios/Goose/`]
- **What it talks to**: the **goosed internal REST+SSE API**, *not* ACP. Evidence — `goose-ios/Goose/GooseAPIService.swift`: `POST {baseURL}/reply` SSE streaming, `X-Secret-Key` header on every call, plus `/agent/start`, `/agent/resume`, `/agent/update_from_session`, `/agent/update_provider`, `/agent/add_extension`, `/sessions`, `/sessions/insights`, config read/write endpoints. This is goosed's private HTTP surface — the protocol goose is migrating *away* from (desktop app included).
- **Transport to reach goosed**: (a) **lapstone-tunnel** — a Cloudflare Durable Object websocket tunnel (`michaelneale/lapstone-tunnel`, worker `cloudflare-tunnel-proxy.michael-neale.workers.dev`, outbound ws client exposes an HTTPS URL behind NAT; "public shared tunnel service - best effort only") — still the *default* in `goose-ios/README.md` and `launch_tunnel.sh`; (b) **Tailscale** as the alternative (option chosen at launch, `launch_tailscale.sh`); (c) a demo goosed on fly.io as the zero-config trial default. Pairing is a QR deep link `goosechat://configure?data={url,secret}`.
- **Maturity**: shipped — App Store listing "Goose AI" (id6752889295, free, iOS 17+, seller Michael Neale, v1.1.6 dated 16 Mar) plus a TestFlight public beta; App Store page live as of 2026-09-19.
- **Last activity**: 2026-03-16 ("fix: forward-compatible SSE decoding for goosed API changes" #163; before that the 2026-01 migration and resume-endpoint fix #161). No ACP work.
- **Note**: the README's own roadmap says "We plan to migrate the client to use a http/remote version of the ACP" — i.e. ACP is explicitly *not done* here.

### 2.2 `goose-android-agent/` — on-device Android agent experiment (not a remote client)

- **Stack**: Kotlin/Gradle Android app (`build.gradle.kts`, `app/src/main/java/xyz/block/gosling/`); Firebase App Distribution for prebuilt APKs; CI on main (`android-ci.yml`, `android-release.yml`).
- **What it is**: a **full agent that runs on the phone** (accessibility-driven device automation, notification reaction, home-screen replacement, calendar/contacts tools) — the opposite of a remote client. Self-described "experimental project… requires deep access to your device, use at own risk"; "a research project, not for production use" (`goose-android-agent/README.md`).
- **Transport**: none to a desktop — the agent is local; it proposes "**mobile MCP**" (mMCP), a broadcast-receiver contract for other Android apps to expose tools; spec draft lives on branch `micn/mmcp-draft` (`mMCP-spec/SPECIFICATION.md`); example app `michaelneale/breezy-weather`.
- **Maturity**: PoC; last activity 2026-03-16 ("android: Add basic tests for features package" #158).
- **The "Android client" is NOT here**: the README roadmap points to a planned `goose-android` *client* port of goose-ios, "a very early start… at https://github.com/michaelneale/goose-android — help wanted". No archived Android client inside this repo.

### 2.3 `mobile-web/` — pure-browser goose-roam web client (the live front)

- **Where it lives**: branch `mobile-web-client` via **PR #171** ("feat(mobile-web): browser roam web client extracted from goose PR #10537", created 2026-08-20, open); the *built site is already deployed and live* from the `roam-web` branch to `https://aaif-goose.github.io/goose-mobile/` (Pages API confirms branch source; roam-web tip 2026-08-11). PR #171 says the desktop pairing QR deep-links to that exact URL, and proposes a follow-up PR on goose #10537 to delete `crates/goose-roaming/web/**` from the goose repo.
- **Provenance**: extracted from `aaif-goose/goose` PR #10537 ("Roaming agents: p2p agent sharing over iroh") — which is now **CLOSED without merge** in the goose repo; the host side (`crates/goose-roaming`, `goose roam` CLI) *is* merged in upstream goose main (verified in `aaif-goose/goose` contents), and this repo's own tree agrees with that split.
- **Stack**: Rust→WASM crate `goose-roaming-web` (iroh 1.0.2 compiled to `wasm32-unknown-unknown`; QUIC-over-WebSocket relay transport, roam handshake, card decode — ~400 lines) + a Vite/React/TypeScript webapp (~450 lines) that **vendors goose's real desktop components** (MarkdownContent, ToolCallStatusIndicator, desktop Tailwind theme, 40-file import closure) and **`GooseClient` from the goose SDK** (TS), per `mobile-web/README.md`.
- **Transport(s)**, from source:
  1. **`goose roam` p2p pairing over iroh**: the browser tab is itself a roam peer; `goose+roam://` host cards, managed iroh relays (mesh-llm's per-region `iroh.link` relays, open registration, QUIC tunneled over `wss://`), end-to-end encrypted; `roam-stream.ts` adapts the wasm duplex to Web Streams for the ACP SDK's `ndJsonStream`, and `ClientSideConnection` (agent-client-protocol TS SDK 0.19.0 at the time of the README) drives ACP over it.
  2. **ACP Streamable HTTP at `/acp`**: the vendored SDK ships `webapp/src/vendor/goose-sdk/http-stream.ts` — a complete ACP Streamable HTTP client (connection-scoped GET SSE + session-scoped SSE streams on `{base}/acp`, `Acp-Connection-Id`/`Acp-Session-Id` headers). The current `App.tsx` wiring is roam-only (`App({ roam }: { roam: RoamClient })`), but the HTTP `/acp` transport is already in the tree via `GooseClient`.
- **Maturity**: "proven end to end" per `mobile-web/README.md` + `webapp/tests/e2e.mjs` (headless Chrome ↔ managed relay ↔ `goose roam share`, prompt in → live agent response); PR-run CI builds wasm + typechecks (`mobile-web-deploy.yml`). Desktop-parity UI is inherited by construction (vendored desktop componentry + Electron-surface shim in `shim.ts`).
- **Known caveats carried in its own README**: browser key in localStorage is a remote-admin credential (needs dedicated origin + CSP before public deploy); version-skew/reconnect semantics unresolved; relay namespace is personal today ("a real deployment wants a goose/Block-owned relay namespace with an SLA").

### 2.4 Non-artifacts checked

- **PR #172** ("Add Build Remote Agent phone pairing (gbr/1)", open, updated 2026-09-07): a README-only note tying goose-ios spectating to a third-party product (`grokbuildremote.com`, Linespotting AB); explicitly "does not replace the tunnel"; not a base for anything.
- **`micn/mmcp-client`**, `micn/mmcp-draft`, and the other `micn/*`, `wes/*`, `cut-tokens`, `home-screen`, `notification-handling` branches: March–April 2025 experiments (Android mMCP drafts, formatting/keyboard fixes) — stale.
- **Old transports**: the Cloudflare lapstone-tunnel is *not* retired in-repo (still the default path in goose-ios README/scripts; PR #172 reaffirms "does not replace the tunnel"), but it is iOS-app-scoped legacy infrastructure, not part of any ACP story.

## 3. The destination baseline (verified)

"Current goose serve — v1.51.x — speaks standard ACP V1 JSON-RPC at `/acp` with `X-Secret-Key`/`?token=` auth, official agent-client-protocol SDK 2.0" — confirmed in `PhillipChaffee/goose` (mirror of upstream main) and upstream releases:

- Workspace version `1.51.0` (`Cargo.toml`); upstream goose released **v1.51.0 on 2026-09-17** (`gh release list -R aaif-goose/goose`).
- `agent-client-protocol = "2.0.0"` (rust-sdk rev `c97a5203`) in `Cargo.toml`/`Cargo.lock`.
- `crates/goose/src/acp/transport/auth.rs`: token accepted from `X-Secret-Key` header **or** `?token=` query param; tests in `crates/goose/tests/acp_transport_auth_test.rs`.
- `goose roam` host side is merged and current upstream: `crates/goose-roaming/` + `crates/goose-cli/src/commands/{roam,roam_client,roam_full_bridge,roam_proxy}.rs` present in `aaif-goose/goose` main; local tree also carries `crates/goose-roaming/tests/{end_to_end,path_upgrade}.rs`.
- Desktop TS side uses `@agentclientprotocol/sdk` 1.3.0 (`ui/desktop/package.json`) — a separate SDK line from the Rust 2.0 crate; the vendored web client pinned 0.19.0 when extracted.

## 4. Recommendation

**Base workstream one on the `mobile-web/` browser roam client (PR #171 + the live `roam-web` Pages deployment).** It is the only artifact that already speaks the destination protocol surface — ACP over both the `goose roam`/iroh duplex and a vendored ACP Streamable HTTP `/acp` stream (`GooseClient` + `http-stream.ts`) — it renders with goose's real desktop components (desktop-parity UI is inherited, not rebuilt), it is the only artifact with development activity in the last five months (Aug 2026), and it is platform-neutral: the React/TS app is the natural codebase to wrap for iOS+Android store distribution (e.g. Capacitor) or to ship as a PWA, and the host side (`goose roam` + `goose serve /acp`) is already merged in the goose v1.51.x the effort targets.

**Runner-up: `goose-ios`.** What would have to be true to switch to it:

1. The transport bar closes: it would need a full client-protocol rewrite in Swift from goosed's private REST+SSE API to standard ACP `/acp` (the README itself lists this as unstarted roadmap work).
2. An Android sibling becomes real: today there is no Android client in-repo — only the on-device agent experiment and a "help wanted" pointer to `michaelneale/goose-android`; a second, independent native port would be required.
3. A decision to accept losing the shared-UI-desktop-parity property: parity would have to be re-implemented twice, in Swift and Kotlin.

In other words, switching to goose-ios means paying the two costs (ACP rewrite + Android rewrite) that the web client avoids; conversely, the web client's cost is a native shell plus hardening of the items its own README flags (key storage/CSP, version skew, reconnect semantics).

## 5. Open items this survey did not settle

- Whether upstream intends to keep `goose roam` relays' open registration (the web client depends on `AccessConfig::Everyone` relays for browsers, which cannot set WebSocket auth headers).
- Fate of goose PR #10537's follow-up (deleting `crates/goose-roaming/web/**` from the goose repo) — PR #171 proposes it; goose repo state should be re-checked before workstream one starts.
- The iOS App Store app's provider story (issue #164 "Stuck on loading a session 85% of times", open since 2026-04-08, suggests active-but-minor iOS maintenance).

### Source index

- `gh api repos/aaif-goose/goose-mobile` (metadata), `/contents`, `/branches`, `/commits?per_page=30`, `?path=goose-ios|goose-android-agent`, `/pages`, `/compare/...`, issues/PRs #156–#172.
- Raw sources: `README.md` (root, main), `goose-ios/README.md`, `goose-ios/AGENTS.md`, `goose-ios/launch_tunnel.sh`, `goose-ios/launch_goosed.sh`, `goose-ios/Goose/GooseAPIService.swift`, `goose-android-agent/README.md`, branch `mobile-web-client`: `mobile-web/README.md`, `mobile-web/webapp/{README.md,TRYME.md,src/{App,ConnectPanel,hosts,roam-stream,shim}.tsx|.ts}`, `webapp/src/vendor/goose-sdk/http-stream.ts`, `.github/workflows/mobile-web-deploy.yml`.
- Local goose repo (`PhillipChaffee/goose`, 1.51.0): `Cargo.toml`, `Cargo.lock`, `crates/goose/src/acp/transport/auth.rs`, `crates/goose/tests/acp_transport_auth_test.rs`, `crates/goose-cli/src/commands/`, `ui/desktop/package.json`; upstream `aaif-goose/goose` contents + releases; PR `aaif-goose/goose#10537`.
- App Store listing `apps.apple.com/au/app/goose-ai/id6752889295` (fetched 2026-09-19); `block.github.io/goose/docs/experimental/goose-mobile/` (404).
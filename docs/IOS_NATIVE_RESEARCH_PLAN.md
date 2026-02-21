# Native iOS Strategy for iRonClaw (IronClaw) — Research & Execution Plan

## Scope and assumptions

This plan is **research-only** and does not introduce runtime code changes. It maps viable paths to a native iOS app using the current IronClaw architecture as the backend/control plane.

## What exists today (relevant to iOS)

- IronClaw already exposes a substantial gateway API surface (chat, memory, jobs, logs, routines, skills, settings, pairing) with bearer-token auth.
- Real-time updates are available via both SSE and WebSocket.
- There is an OpenAI-compatible API (`/v1/chat/completions`, `/v1/models`) that allows standard clients to connect with base URL + bearer token.
- The project is explicitly local-first and currently tuned for localhost/LAN style usage patterns.

## Candidate architecture options

### Option A — Native iOS app as a thin client to IronClaw Gateway (recommended first)

**Pattern**
- SwiftUI app talks to a running IronClaw gateway on LAN/localhost-over-tunnel.
- Primary integration uses `/api/chat/*` + `/api/chat/events` (or `/api/chat/ws`) for rich UX.

**Pros**
- Fastest path to production-quality native UI.
- Reuses current backend behavior and security model.
- Low maintenance vs embedding Rust runtime into iOS.

**Cons**
- Requires a reachable gateway host.
- Offline-on-phone without host is not supported.

**When to choose**
- You want a reliable native iOS client quickly.
- You can accept “phone app as controller/client” for v1.

---

### Option B — OpenAI-compatible iOS client mode

**Pattern**
- iOS app uses OpenAI SDK-compatible networking against IronClaw’s `/v1/*` routes.

**Pros**
- Fast integration using existing OpenAI client abstractions.
- Good for experimentation and broad model compatibility.

**Cons**
- Loses some IronClaw-specific UX semantics (threads/jobs/approvals/pairing/events) unless additional custom endpoints are used.

**When to choose**
- You want the quickest prototype while minimizing custom protocol work.

---

### Option C — Hybrid: native iOS + embedded on-device Rust server (Litter-style inspiration)

**Pattern**
- Bundle an iOS-compatible Rust core (xcframework/staticlib) and run local server/bridge on-device.
- Optionally support remote mode and on-device mode toggle.

**Pros**
- Potential offline/standalone operation.
- Strong product differentiation if stable.

**Cons**
- Highest complexity (cross-compilation, FFI boundaries, iOS lifecycle/background constraints, footprint, battery/perf tuning).
- Largest long-term maintenance burden.

**When to choose**
- After thin-client v1 is stable and usage validates demand for full on-device execution.

---

### Option D — Full custom native stack without gateway protocol reuse

**Pattern**
- Reimplement large portions of orchestration/protocol directly for iOS.

**Pros**
- Maximum control.

**Cons**
- Reinvents already-working gateway contracts and security controls.
- Slowest, riskiest, and least leverage.

**When to choose**
- Generally avoid unless strategic constraints require it.

## Recommendation

1. **Phase 1 (Now): Option A + selective Option B compatibility**
   - Build a native SwiftUI app against current gateway APIs.
   - Use SSE first (simpler), then add WebSocket for richer bidirectional control.
   - Keep OpenAI-compatible mode as optional transport for quick interop.
2. **Phase 2 (After PMF): Evaluate Option C**
   - Spike embedded Rust/on-device runtime only after v1 UX + stability goals are met.

## Why this is the best path

- It aligns with current IronClaw strengths (existing authenticated control plane + event streaming).
- It minimizes time-to-value while preserving an upgrade path to on-device execution.
- It avoids premature coupling to a complex mobile runtime bridge before core UX is validated.

## iOS v1 capability map (must-have vs later)

### Must-have for v1
- Token-based connection setup and secure persistence in iOS Keychain.
- Chat send + streamed responses.
- Thread list/history/new thread.
- Approval flows (`approval_needed` -> approve/deny).
- Basic connection diagnostics (health/status).

### Should-have shortly after
- Jobs visibility and prompt/cancel controls.
- Memory read/search.
- Settings read/update for mobile-safe keys.

### Later
- Extensions/skills management UI.
- Full routines management UI.
- Advanced pairing/admin workflows.

## Proposed technical blueprint (Option A)

### Mobile architecture
- **SwiftUI + The Composable Architecture (or MVVM)** for deterministic state handling.
- **Networking layer**:
  - REST client for command/query endpoints.
  - SSE client for event stream (`/api/chat/events?token=...`).
  - Optional WebSocket client for unified bi-directional channel.
- **Security**:
  - Store bearer token in Keychain.
  - Never log token.
  - TLS required off-localhost deployments.

### Backend integration priorities
1. `/api/health`
2. `/api/gateway/status`
3. `/api/chat/send`
4. `/api/chat/events` (SSE)
5. `/api/chat/history`, `/api/chat/threads`, `/api/chat/thread/new`
6. `/api/chat/approval`
7. `/api/chat/ws` (optional second step)

## Gaps / risks identified from current parity matrix

- iOS-specific APNs wake pipeline is currently not implemented.
- Device pairing is not implemented.
- Some event-broadcast wiring is partial and should be validated for mobile UX.
- Network mode capabilities are partial; remote-hardening plan is needed for non-LAN usage.

## 6-week execution plan

### Week 1 — Discovery & contract freeze
- Record endpoint contract assumptions for iOS v1.
- Validate event ordering and reconnection behavior under flaky networks.
- Define UX for token onboarding and thread model.

### Week 2 — Foundation
- Build app shell, auth flow, Keychain storage, environment profiles (local/LAN/remote).
- Implement health/status checks.

### Week 3 — Core chat
- Implement send + streamed assistant responses via SSE.
- Thread list/history/new thread with pagination handling.

### Week 4 — Control features
- Implement approval actions and robust error surfaces.
- Add basic jobs visibility.

### Week 5 — Hardening
- Reconnect logic, background/foreground transitions, retry/backoff.
- Security pass for token handling and transport assumptions.

### Week 6 — Beta readiness
- Test matrix (simulator/device, Wi-Fi changes, gateway restarts).
- Instrumentation and bug triage.
- Publish internal TestFlight candidate.

## Decision framework for revisiting on-device runtime (Option C)

Proceed to embedded runtime spike only if:
- v1 client DAU and retention justify offline/standalone demand.
- Memory/CPU budget targets for sustained on-device operation are feasible.
- Team accepts ongoing Rust+iOS bridge maintenance cost.

If greenlit, run a 2-week spike:
- Rust iOS cross-compile PoC.
- Minimal FFI surface (start/stop/status) + local loopback API smoke test.
- Battery/perf baseline on at least 2 physical devices.

## Notes on Litter (Codex iOS) as reference

Litter demonstrates a practical dual-mode model:
- remote-only iOS client mode,
- plus optional bundled on-device Rust bridge mode.

This is useful as **architecture inspiration** (mode toggles, bridge isolation), not a direct implementation dependency.

## Skill discovery notes

Searched the open skills ecosystem for SwiftUI/iOS focused skills via:
- `npx skills find swiftui`
- `npx skills find "ios swift"`

No matching skills were returned in this environment at research time, so this plan is based on repository + architecture analysis directly.

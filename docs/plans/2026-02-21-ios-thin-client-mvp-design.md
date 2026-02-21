# iOS Thin Client MVP Design (Gateway-Direct)

Date: 2026-02-21  
Status: Approved for planning  
Decision: Proceed with Option A (Native iOS app as a thin client to IronClaw Gateway)

## 1. Goals and constraints

### Goals
- Deliver a public-ready native iOS MVP.
- Support both LAN and remote connectivity from day one.
- Ship core chat plus jobs and memory capabilities.

### Constraints
- Minimal backend changes (critical fixes only).
- Reuse existing gateway APIs and auth model.
- No on-device embedded runtime in MVP.

## 2. Selected approach

### Chosen
- **Gateway-Direct Client**: iOS app talks directly to existing IronClaw `/api/*` endpoints.
- Use SSE as primary realtime transport for MVP.
- Keep WebSocket as an enhancement path after baseline stability.

### Why this approach
- Fastest path under minimal-backend-change constraint.
- Preserves IronClaw-specific UX semantics (threads, approvals, jobs, memory).
- Lowest implementation and maintenance risk for v1.

### Alternatives considered
1. Hybrid transport (`/v1/*` for chat + `/api/*` for everything else): split semantics and higher edge-case complexity.
2. Mobile BFF adapter layer: cleaner mobile protocol but increases backend surface and maintenance burden.

## 3. Architecture baseline

### System shape
- Native SwiftUI iOS app is a thin client.
- IronClaw Gateway remains control plane and execution plane.
- No embedded Rust runtime on device in MVP.

### Transport
- REST for command/query endpoints.
- SSE for assistant streaming and approval events.
- Optional WebSocket later if needed.

### Connectivity profiles
- LAN profile: local network gateway.
- Remote profile: internet-reachable TLS endpoint.
- Per-profile token stored in iOS Keychain.

### Minimal backend change policy
- Consume existing contracts by default.
- Backend updates only when correctness, security, or reliability are blocked.

## 4. MVP capability scope

### In scope (MVP)
- Connection onboarding with profile + token verification.
- Core chat:
  - Send message
  - Streamed assistant response
  - Thread list/history/new thread
  - Approval flows (approve/deny)
- Jobs:
  - List jobs
  - Job details
  - Cancel action
- Memory:
  - Search
  - Read details
- Diagnostics:
  - Health/status checks
  - Connectivity state and actionable failure messages

### Out of scope (MVP)
- APNs wake pipeline.
- Device pairing workflows.
- Share sheet, voice input, background listening.
- Advanced routines/skills administration.
- Full WebSocket-first UX.

## 5. iOS app component map

- `ConnectionManager`
  - Profile storage, active profile selection, reachability state.
- `AuthStore`
  - Keychain token persistence, secure update/clear paths.
- `GatewayClient`
  - Typed REST client, SSE stream client, shared auth and error mapping.
- `ChatModule`
  - Threads/history/send/stream render/approval actions.
- `JobsModule`
  - List/detail/cancel + active-view polling fallback.
- `MemoryModule`
  - Search/read UI and result grouping.
- `DiagnosticsModule`
  - Gateway health, auth state, recent error metadata.

## 6. Data flow design

### Chat flow
1. User selects LAN or remote profile.
2. App validates token using authenticated endpoint.
3. App loads threads/history.
4. User sends message to `/api/chat/send`.
5. SSE events stream assistant output and state transitions.
6. Approval-needed event triggers approve/deny action via `/api/chat/approval`.
7. Stream continues to completion.

### Jobs flow
1. Load jobs list on module open.
2. Poll while module is active and app is foregrounded.
3. Cancel triggers API call and immediate list refresh.

### Memory flow
1. Search query submitted to memory search endpoint.
2. Result selection loads memory read/details endpoint.

## 7. Reliability and error handling

### Reliability strategy
- SSE auto-reconnect with exponential backoff + jitter.
- On reconnect, fetch latest active thread history to self-heal missed events.
- Pause expensive loops in background; reconcile on foreground resume.
- Idempotent reducer behavior for duplicate fragments/events after reconnect.

### Error handling contract
- Auth errors (401/403): block protected actions, preserve drafts, prompt token refresh.
- Network errors: retryable states with clear connection diagnostics.
- Server errors (5xx): user-visible failure state + diagnostic metadata.
- Stream failures: automatic reconnect, then reconciliation fetch.

## 8. Public-ready quality gates

1. Connectivity matrix passes for both LAN and remote profiles on physical devices.
2. Lifecycle matrix passes: background/foreground, app restart, gateway restart.
3. Security checks pass: token never logged, Keychain-only storage, TLS enforced remotely.
4. Usability checks pass: onboarding is recoverable and no dead-end states remain.
5. Stability checks pass: crash-free and reconnect success targets met for release candidate.

## 9. Milestone-based process map (no calendar timeline)

### Milestone A: Contract lock
- Freeze endpoint payload expectations for iOS MVP.
- Define strict criteria for what qualifies as a critical backend fix.

### Milestone B: Foundation
- App shell, connection profiles, token onboarding, diagnostics baseline, typed API client.

### Milestone C: Core chat
- Threads/history/new thread/send.
- SSE streaming and approval flows.
- Reconnect reconciliation behavior.

### Milestone D: Jobs + memory
- Jobs list/detail/cancel.
- Memory search/read with robust empty/error states.

### Milestone E: Public-ready hardening
- Reliability tuning and instrumentation.
- Full QA matrix and bug triage.
- TestFlight release candidate.

### Milestone F: App Store submission readiness
- Final regression checks.
- Policy/privacy and operational support artifacts.

## 10. Critical backend fixes policy (guardrail)

Only perform backend work for:
- Contract ambiguity that blocks stable mobile integration.
- Security defects affecting token handling or remote access posture.
- Event consistency defects that break reconnection correctness.

All non-critical backend enhancements defer until after MVP release readiness.

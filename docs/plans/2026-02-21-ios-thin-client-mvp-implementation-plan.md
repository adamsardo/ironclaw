# iOS Thin Client MVP Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Build a public-ready native iOS thin client (LAN + remote profiles) for IronClaw Gateway with core chat, jobs visibility/cancel, memory search/read, and diagnostics.

**Architecture:** Keep Rust gateway as the control plane and implement a SwiftUI iOS app that consumes existing `/api/*` endpoints with bearer auth. Use SSE as the primary realtime transport; perform reconnect + history reconciliation client-side. Backend changes are allowed only when contract tests prove a critical correctness/security/reliability gap.

**Tech Stack:** Rust (`cargo test`) for gateway contract tests; Swift 5.10+, SwiftUI, URLSession, async/await, XCTest, XcodeGen, xcodebuild.

---

## Execution guardrails

- Required supporting skills during execution: `@test-driven-development`, `@systematic-debugging`, `@verification-before-completion`, `@requesting-code-review`.
- Keep tasks DRY/YAGNI: no APNs, no pairing UX, no voice/share-sheet, no embedded runtime.
- If any behavior in `FEATURE_PARITY.md` changes, update `FEATURE_PARITY.md` in the same commit.

## Pre-flight (run once before Task 1)

Run:
```bash
xcode-select -p
swift --version
cargo --version
which xcodegen || brew install xcodegen
```

Expected:
- Xcode command line tools path is printed.
- Swift and Cargo versions are printed.
- `xcodegen` exists after install.

---

### Task 1: Lock Gateway API Contract Tests for iOS

**Files:**
- Create: `tests/ios_gateway_contract_integration.rs`
- Modify: `Cargo.toml` (only if a new dev dependency is required)
- Test: `tests/ios_gateway_contract_integration.rs`

**Step 1: Write the failing test**

```rust
#[tokio::test]
async fn test_ios_gateway_status_contract() {
    let (addr, _state, _rx) = start_ios_contract_server().await;
    let client = reqwest::Client::new();

    let resp = client
        .get(format!("http://{addr}/api/gateway/status"))
        .bearer_auth("test-token-12345")
        .send()
        .await
        .unwrap();

    assert_eq!(resp.status(), reqwest::StatusCode::OK);
    let payload: serde_json::Value = resp.json().await.unwrap();
    assert!(payload.get("uptime_secs").is_some());
    assert!(payload.get("sse_connections").is_some());
    assert!(payload.get("ws_connections").is_some());
}
```

**Step 2: Run test to verify it fails**

Run: `cargo test --test ios_gateway_contract_integration test_ios_gateway_status_contract -q`
Expected: FAIL with compile error for missing `start_ios_contract_server`.

**Step 3: Write minimal implementation**

```rust
async fn start_ios_contract_server(
) -> (
    std::net::SocketAddr,
    std::sync::Arc<ironclaw::channels::web::server::GatewayState>,
    tokio::sync::mpsc::Receiver<ironclaw::channels::IncomingMessage>,
) {
    // Copy the helper pattern from tests/ws_gateway_integration.rs
    // using auth token: "test-token-12345" and start_server(addr, state, token).
    unimplemented!()
}
```

Add additional contract tests in the same file for:
- `GET /api/health`
- `GET /api/chat/threads`
- `GET /api/chat/history?limit=50`
- `POST /api/memory/search`
- `GET /api/memory/read?path=...`
- `GET /api/jobs`
- `POST /api/chat/approval` (invalid UUID must return `400`)

**Step 4: Run test to verify it passes**

Run: `cargo test --test ios_gateway_contract_integration -q`
Expected: PASS for all iOS contract tests.

**Step 5: Commit**

```bash
git add tests/ios_gateway_contract_integration.rs Cargo.toml
git commit -m "test: add iOS gateway contract integration tests"
```

---

### Task 2: Bootstrap Shared Swift Client Package

**Files:**
- Create: `apps/ios/IronClawClientKit/Package.swift`
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Models/ConnectionProfile.swift`
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Models/GatewayEnvironment.swift`
- Test: `apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/ConnectionProfileTests.swift`

**Step 1: Write the failing test**

```swift
import XCTest
@testable import IronClawClientKit

final class ConnectionProfileTests: XCTestCase {
    func test_defaultLanProfileUsesHttp() {
        let profile = ConnectionProfile.defaultLAN(host: "192.168.1.10", port: 3000)
        XCTAssertEqual(profile.baseURL.absoluteString, "http://192.168.1.10:3000")
        XCTAssertEqual(profile.transport, .sse)
    }
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter ConnectionProfileTests/test_defaultLanProfileUsesHttp`
Expected: FAIL with "no such module 'IronClawClientKit'" or missing `ConnectionProfile`.

**Step 3: Write minimal implementation**

```swift
public enum StreamTransport: String, Codable {
    case sse
}

public struct ConnectionProfile: Codable, Equatable, Identifiable {
    public let id: UUID
    public var name: String
    public var baseURL: URL
    public var transport: StreamTransport

    public static func defaultLAN(host: String, port: Int) -> ConnectionProfile {
        ConnectionProfile(
            id: UUID(),
            name: "LAN",
            baseURL: URL(string: "http://\(host):\(port)")!,
            transport: .sse
        )
    }
}
```

**Step 4: Run test to verify it passes**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter ConnectionProfileTests`
Expected: PASS.

**Step 5: Commit**

```bash
git add apps/ios/IronClawClientKit
git commit -m "feat(ios): bootstrap IronClawClientKit package"
```

---

### Task 3: Add Token + Profile Persistence Layer

**Files:**
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Auth/AuthTokenStore.swift`
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Connection/ProfileStore.swift`
- Test: `apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/ProfileStoreTests.swift`
- Test: `apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/AuthTokenStoreTests.swift`

**Step 1: Write the failing test**

```swift
func test_profileStoreTracksActiveProfile() throws {
    let store = InMemoryProfileStore()
    let lan = ConnectionProfile.defaultLAN(host: "127.0.0.1", port: 3000)
    try store.saveProfiles([lan], activeProfileID: lan.id)

    let loaded = try store.loadProfiles()
    XCTAssertEqual(loaded.activeProfileID, lan.id)
    XCTAssertEqual(loaded.profiles.count, 1)
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter ProfileStoreTests/test_profileStoreTracksActiveProfile`
Expected: FAIL with missing `InMemoryProfileStore`.

**Step 3: Write minimal implementation**

```swift
public struct LoadedProfiles {
    public var profiles: [ConnectionProfile]
    public var activeProfileID: UUID?
}

public protocol ProfileStore {
    func loadProfiles() throws -> LoadedProfiles
    func saveProfiles(_ profiles: [ConnectionProfile], activeProfileID: UUID?) throws
}

public final class InMemoryProfileStore: ProfileStore {
    private var state = LoadedProfiles(profiles: [], activeProfileID: nil)
    public init() {}

    public func loadProfiles() throws -> LoadedProfiles { state }
    public func saveProfiles(_ profiles: [ConnectionProfile], activeProfileID: UUID?) throws {
        state = LoadedProfiles(profiles: profiles, activeProfileID: activeProfileID)
    }
}
```

Implement `KeychainTokenStore` with protocol-first API so tests can run using an in-memory fake.

**Step 4: Run test to verify it passes**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter ProfileStoreTests`
Expected: PASS.

Run: `swift test --package-path apps/ios/IronClawClientKit --filter AuthTokenStoreTests`
Expected: PASS.

**Step 5: Commit**

```bash
git add apps/ios/IronClawClientKit/Sources/IronClawClientKit/Auth apps/ios/IronClawClientKit/Sources/IronClawClientKit/Connection apps/ios/IronClawClientKit/Tests/IronClawClientKitTests
git commit -m "feat(ios): add profile and token persistence abstractions"
```

---

### Task 4: Implement Typed Gateway HTTP Client

**Files:**
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Networking/GatewayAPIClient.swift`
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Networking/GatewayEndpoint.swift`
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Networking/GatewayDTOs.swift`
- Test: `apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/GatewayAPIClientTests.swift`

**Step 1: Write the failing test**

```swift
func test_sendAddsBearerTokenAndThreadID() async throws {
    let transport = URLProtocolTestTransport()
    let client = GatewayAPIClient(baseURL: URL(string: "http://127.0.0.1:3000")!, token: "abc", transport: transport)

    _ = try await client.sendMessage(content: "hello", threadID: "t1")

    let req = try XCTUnwrap(transport.lastRequest)
    XCTAssertEqual(req.value(forHTTPHeaderField: "Authorization"), "Bearer abc")
    XCTAssertEqual(req.url?.path, "/api/chat/send")
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter GatewayAPIClientTests/test_sendAddsBearerTokenAndThreadID`
Expected: FAIL with missing `GatewayAPIClient`.

**Step 3: Write minimal implementation**

```swift
public final class GatewayAPIClient {
    public init(baseURL: URL, token: String, transport: HTTPTransport = URLSessionTransport()) {
        self.baseURL = baseURL
        self.token = token
        self.transport = transport
    }

    public func sendMessage(content: String, threadID: String?) async throws -> SendMessageResponse {
        var req = URLRequest(url: baseURL.appending(path: "/api/chat/send"))
        req.httpMethod = "POST"
        req.setValue("Bearer \(token)", forHTTPHeaderField: "Authorization")
        req.setValue("application/json", forHTTPHeaderField: "Content-Type")
        req.httpBody = try JSONEncoder().encode(["content": content, "thread_id": threadID])
        return try await transport.perform(req, as: SendMessageResponse.self)
    }

    private let baseURL: URL
    private let token: String
    private let transport: HTTPTransport
}
```

Add methods for:
- `fetchThreads()`
- `fetchHistory(threadID:limit:before:)`
- `sendApproval(requestID:action:threadID:)`
- `fetchJobs()` / `fetchJobDetail(id:)` / `cancelJob(id:)`
- `searchMemory(query:limit:)` / `readMemory(path:)`
- `fetchGatewayStatus()` / `fetchHealth()`

**Step 4: Run test to verify it passes**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter GatewayAPIClientTests`
Expected: PASS.

**Step 5: Commit**

```bash
git add apps/ios/IronClawClientKit/Sources/IronClawClientKit/Networking apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/GatewayAPIClientTests.swift
git commit -m "feat(ios): implement typed gateway API client"
```

---

### Task 5: Implement SSE Decoder + Reconnect Policy

**Files:**
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Streaming/SSEEvent.swift`
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Streaming/SSEStreamDecoder.swift`
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Streaming/ReconnectPolicy.swift`
- Test: `apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/SSEStreamDecoderTests.swift`
- Test: `apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/ReconnectPolicyTests.swift`

**Step 1: Write the failing test**

```swift
func test_decodesStreamChunkEvent() throws {
    let raw = "event: stream_chunk\ndata: {\"type\":\"stream_chunk\",\"content\":\"hel\",\"thread_id\":\"t1\"}\n\n"
    let decoded = try SSEStreamDecoder().decode(raw)
    XCTAssertEqual(decoded.count, 1)
    XCTAssertEqual(decoded.first, .streamChunk(content: "hel", threadID: "t1"))
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter SSEStreamDecoderTests/test_decodesStreamChunkEvent`
Expected: FAIL with missing `SSEStreamDecoder`.

**Step 3: Write minimal implementation**

```swift
public enum SSEEvent: Equatable {
    case streamChunk(content: String, threadID: String?)
    case response(content: String, threadID: String)
    case approvalNeeded(requestID: String, toolName: String, description: String, parameters: String, threadID: String?)
    case heartbeat
}

public struct ReconnectPolicy {
    public func delaySeconds(forAttempt attempt: Int) -> TimeInterval {
        let base = min(pow(2.0, Double(attempt)), 30)
        let jitter = Double.random(in: 0...0.3)
        return base + jitter
    }
}
```

Include decoding for `stream_chunk`, `response`, `approval_needed`, `heartbeat`, and `error`.

**Step 4: Run test to verify it passes**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter SSEStreamDecoderTests`
Expected: PASS.

Run: `swift test --package-path apps/ios/IronClawClientKit --filter ReconnectPolicyTests`
Expected: PASS.

**Step 5: Commit**

```bash
git add apps/ios/IronClawClientKit/Sources/IronClawClientKit/Streaming apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/SSEStreamDecoderTests.swift apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/ReconnectPolicyTests.swift
git commit -m "feat(ios): add SSE decoding and reconnect policy"
```

---

### Task 6: Implement Chat Domain Service (Threads, Send, Stream, Approval)

**Files:**
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Features/Chat/ChatService.swift`
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Features/Chat/ChatState.swift`
- Test: `apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/ChatServiceTests.swift`

**Step 1: Write the failing test**

```swift
func test_streamChunkAppendsToPendingAssistantMessage() async throws {
    let service = ChatService.testInstance()
    service.apply(event: .streamChunk(content: "Hello", threadID: "t1"))
    service.apply(event: .streamChunk(content: " world", threadID: "t1"))

    XCTAssertEqual(service.state.pendingAssistantTextByThread["t1"], "Hello world")
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter ChatServiceTests/test_streamChunkAppendsToPendingAssistantMessage`
Expected: FAIL with missing `ChatService`.

**Step 3: Write minimal implementation**

```swift
public final class ChatService {
    public private(set) var state = ChatState()

    public func apply(event: SSEEvent) {
        switch event {
        case let .streamChunk(content, threadID):
            guard let threadID else { return }
            let current = state.pendingAssistantTextByThread[threadID] ?? ""
            state.pendingAssistantTextByThread[threadID] = current + content
        case let .response(content, threadID):
            state.finalResponsesByThread[threadID] = content
            state.pendingAssistantTextByThread[threadID] = nil
        default:
            break
        }
    }
}
```

Add methods to call API client for:
- loading threads/history
- sending message
- approving/denying execution requests
- reconciliation fetch after stream reconnect

**Step 4: Run test to verify it passes**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter ChatServiceTests`
Expected: PASS.

**Step 5: Commit**

```bash
git add apps/ios/IronClawClientKit/Sources/IronClawClientKit/Features/Chat apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/ChatServiceTests.swift
git commit -m "feat(ios): implement chat service and stream state reducer"
```

---

### Task 7: Implement Jobs Domain Service

**Files:**
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Features/Jobs/JobsService.swift`
- Test: `apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/JobsServiceTests.swift`

**Step 1: Write the failing test**

```swift
func test_cancelJobRefreshesJobList() async throws {
    let api = MockGatewayAPIClient()
    api.jobs = [.init(id: "j1", title: "Run", state: "in_progress")]
    let service = JobsService(api: api)

    try await service.cancel(jobID: "j1")

    XCTAssertTrue(api.cancelCalled)
    XCTAssertEqual(service.jobs.first?.id, "j1")
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter JobsServiceTests/test_cancelJobRefreshesJobList`
Expected: FAIL with missing `JobsService`.

**Step 3: Write minimal implementation**

```swift
public final class JobsService {
    public private(set) var jobs: [JobDTO] = []
    private let api: GatewayAPIClientProtocol

    public init(api: GatewayAPIClientProtocol) { self.api = api }

    public func refresh() async throws {
        jobs = try await api.fetchJobs()
    }

    public func cancel(jobID: String) async throws {
        _ = try await api.cancelJob(id: jobID)
        try await refresh()
    }
}
```

**Step 4: Run test to verify it passes**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter JobsServiceTests`
Expected: PASS.

**Step 5: Commit**

```bash
git add apps/ios/IronClawClientKit/Sources/IronClawClientKit/Features/Jobs apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/JobsServiceTests.swift
git commit -m "feat(ios): implement jobs list/detail/cancel service"
```

---

### Task 8: Implement Memory Domain Service

**Files:**
- Create: `apps/ios/IronClawClientKit/Sources/IronClawClientKit/Features/Memory/MemoryService.swift`
- Test: `apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/MemoryServiceTests.swift`

**Step 1: Write the failing test**

```swift
func test_searchThenReadReturnsDocumentBody() async throws {
    let api = MockGatewayAPIClient()
    api.searchResults = [.init(path: "notes/today.md", content: "snippet", score: 0.8)]
    api.readResult = .init(path: "notes/today.md", content: "full document", updatedAt: nil)
    let service = MemoryService(api: api)

    let results = try await service.search(query: "today", limit: 10)
    let doc = try await service.read(path: results[0].path)

    XCTAssertEqual(doc.content, "full document")
}
```

**Step 2: Run test to verify it fails**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter MemoryServiceTests/test_searchThenReadReturnsDocumentBody`
Expected: FAIL with missing `MemoryService`.

**Step 3: Write minimal implementation**

```swift
public final class MemoryService {
    private let api: GatewayAPIClientProtocol

    public init(api: GatewayAPIClientProtocol) { self.api = api }

    public func search(query: String, limit: Int) async throws -> [MemorySearchHitDTO] {
        try await api.searchMemory(query: query, limit: limit)
    }

    public func read(path: String) async throws -> MemoryReadDTO {
        try await api.readMemory(path: path)
    }
}
```

**Step 4: Run test to verify it passes**

Run: `swift test --package-path apps/ios/IronClawClientKit --filter MemoryServiceTests`
Expected: PASS.

**Step 5: Commit**

```bash
git add apps/ios/IronClawClientKit/Sources/IronClawClientKit/Features/Memory apps/ios/IronClawClientKit/Tests/IronClawClientKitTests/MemoryServiceTests.swift
git commit -m "feat(ios): implement memory search/read service"
```

---

### Task 9: Scaffold iOS App Host + Onboarding Flow

**Files:**
- Create: `apps/ios/IronClawMobile/project.yml`
- Create: `apps/ios/IronClawMobile/App/IronClawMobileApp.swift`
- Create: `apps/ios/IronClawMobile/App/AppState.swift`
- Create: `apps/ios/IronClawMobile/App/Features/Onboarding/OnboardingView.swift`
- Create: `apps/ios/IronClawMobile/AppTests/AppStateTests.swift`
- Modify: `.gitignore`

**Step 1: Write the failing test**

```swift
import XCTest
@testable import IronClawMobile

final class AppStateTests: XCTestCase {
    func test_onboardingRequiredWhenNoActiveProfile() {
        let state = AppState(activeProfile: nil)
        XCTAssertTrue(state.requiresOnboarding)
    }
}
```

**Step 2: Run test to verify it fails**

Run:
```bash
xcodegen generate --spec apps/ios/IronClawMobile/project.yml
xcodebuild -project apps/ios/IronClawMobile/IronClawMobile.xcodeproj -scheme IronClawMobile -destination 'platform=iOS Simulator,name=iPhone 16' test
```
Expected: FAIL because app files / `AppState` are missing.

**Step 3: Write minimal implementation**

```swift
@MainActor
final class AppState: ObservableObject {
    @Published var activeProfile: ConnectionProfile?

    init(activeProfile: ConnectionProfile?) {
        self.activeProfile = activeProfile
    }

    var requiresOnboarding: Bool { activeProfile == nil }
}
```

`project.yml` must define:
- iOS app target `IronClawMobile`
- unit test target `IronClawMobileTests`
- local package dependency path `../IronClawClientKit`

Also update `.gitignore` with:
```gitignore
# Xcode
DerivedData/
*.xcuserstate
**/*.xcuserdata/
```

**Step 4: Run test to verify it passes**

Run:
```bash
xcodegen generate --spec apps/ios/IronClawMobile/project.yml
xcodebuild -project apps/ios/IronClawMobile/IronClawMobile.xcodeproj -scheme IronClawMobile -destination 'platform=iOS Simulator,name=iPhone 16' test -only-testing:IronClawMobileTests/AppStateTests
```
Expected: PASS for `AppStateTests`.

**Step 5: Commit**

```bash
git add apps/ios/IronClawMobile .gitignore
git commit -m "feat(ios): scaffold iOS app host with onboarding state"
```

---

### Task 10: Implement Chat Tab UI (Send, Stream, Approval)

**Files:**
- Create: `apps/ios/IronClawMobile/App/Features/Chat/ChatViewModel.swift`
- Create: `apps/ios/IronClawMobile/App/Features/Chat/ChatView.swift`
- Create: `apps/ios/IronClawMobile/AppTests/ChatViewModelTests.swift`

**Step 1: Write the failing test**

```swift
func test_approvalEventCreatesPendingApprovalCard() async throws {
    let vm = ChatViewModel.mock()
    vm.handle(event: .approvalNeeded(requestID: "r1", toolName: "shell", description: "run cmd", parameters: "{}", threadID: "t1"))

    XCTAssertEqual(vm.pendingApprovals.count, 1)
    XCTAssertEqual(vm.pendingApprovals.first?.requestID, "r1")
}
```

**Step 2: Run test to verify it fails**

Run:
```bash
xcodebuild -project apps/ios/IronClawMobile/IronClawMobile.xcodeproj -scheme IronClawMobile -destination 'platform=iOS Simulator,name=iPhone 16' test -only-testing:IronClawMobileTests/ChatViewModelTests
```
Expected: FAIL with missing `ChatViewModel`.

**Step 3: Write minimal implementation**

```swift
@MainActor
final class ChatViewModel: ObservableObject {
    @Published var pendingApprovals: [ApprovalCard] = []

    func handle(event: SSEEvent) {
        if case let .approvalNeeded(requestID, toolName, description, parameters, _) = event {
            pendingApprovals.append(
                ApprovalCard(requestID: requestID, toolName: toolName, description: description, parameters: parameters)
            )
        }
    }
}
```

Wire UI actions:
- Send message
- Approve / deny buttons
- Streaming text updates from SSE events

**Step 4: Run test to verify it passes**

Run:
```bash
xcodebuild -project apps/ios/IronClawMobile/IronClawMobile.xcodeproj -scheme IronClawMobile -destination 'platform=iOS Simulator,name=iPhone 16' test -only-testing:IronClawMobileTests/ChatViewModelTests
```
Expected: PASS.

**Step 5: Commit**

```bash
git add apps/ios/IronClawMobile/App/Features/Chat apps/ios/IronClawMobile/AppTests/ChatViewModelTests.swift
git commit -m "feat(ios): add chat tab with streaming and approvals"
```

---

### Task 11: Implement Jobs, Memory, and Diagnostics Tabs

**Files:**
- Create: `apps/ios/IronClawMobile/App/Features/Jobs/JobsViewModel.swift`
- Create: `apps/ios/IronClawMobile/App/Features/Jobs/JobsView.swift`
- Create: `apps/ios/IronClawMobile/App/Features/Memory/MemoryViewModel.swift`
- Create: `apps/ios/IronClawMobile/App/Features/Memory/MemoryView.swift`
- Create: `apps/ios/IronClawMobile/App/Features/Diagnostics/DiagnosticsViewModel.swift`
- Create: `apps/ios/IronClawMobile/App/Features/Diagnostics/DiagnosticsView.swift`
- Create: `apps/ios/IronClawMobile/AppTests/DiagnosticsViewModelTests.swift`

**Step 1: Write the failing test**

```swift
func test_diagnosticsMarksUnauthorizedAsReauthRequired() async throws {
    let vm = DiagnosticsViewModel.mock()
    vm.apply(error: .unauthorized)
    XCTAssertEqual(vm.primaryActionTitle, "Update Token")
}
```

**Step 2: Run test to verify it fails**

Run:
```bash
xcodebuild -project apps/ios/IronClawMobile/IronClawMobile.xcodeproj -scheme IronClawMobile -destination 'platform=iOS Simulator,name=iPhone 16' test -only-testing:IronClawMobileTests/DiagnosticsViewModelTests
```
Expected: FAIL with missing `DiagnosticsViewModel`.

**Step 3: Write minimal implementation**

```swift
enum ConnectivityFailure: Equatable {
    case unauthorized
    case network
    case server
}

@MainActor
final class DiagnosticsViewModel: ObservableObject {
    @Published var primaryActionTitle: String = "Retry"

    func apply(error: ConnectivityFailure) {
        switch error {
        case .unauthorized:
            primaryActionTitle = "Update Token"
        case .network, .server:
            primaryActionTitle = "Retry"
        }
    }
}
```

Also implement:
- Jobs cancel + refresh behavior
- Memory search + read details flow
- Diagnostics cards for `/api/health` and `/api/gateway/status`

**Step 4: Run test to verify it passes**

Run:
```bash
xcodebuild -project apps/ios/IronClawMobile/IronClawMobile.xcodeproj -scheme IronClawMobile -destination 'platform=iOS Simulator,name=iPhone 16' test
```
Expected: PASS for app unit tests.

**Step 5: Commit**

```bash
git add apps/ios/IronClawMobile/App/Features apps/ios/IronClawMobile/AppTests
git commit -m "feat(ios): add jobs, memory, and diagnostics tabs"
```

---

### Task 12: Public-Ready Verification, Documentation, and Parity Update

**Files:**
- Create: `docs/IOS_MVP_RUNBOOK.md`
- Modify: `README.md`
- Modify: `FEATURE_PARITY.md`
- Create: `docs/IOS_QA_MATRIX.md`

**Step 1: Write the failing test/check**

Add a QA script check file:
- Create `scripts/check_ios_mvp.sh` that exits non-zero if required docs or app artifacts are missing.

```bash
#!/usr/bin/env bash
set -euo pipefail

test -f apps/ios/IronClawMobile/project.yml
test -f docs/IOS_MVP_RUNBOOK.md
test -f docs/IOS_QA_MATRIX.md
```

**Step 2: Run check to verify it fails**

Run: `bash scripts/check_ios_mvp.sh`
Expected: FAIL before docs are added.

**Step 3: Write minimal implementation**

Document and update:
- `docs/IOS_MVP_RUNBOOK.md`
  - onboarding steps
  - LAN profile setup
  - remote profile/TLS setup
  - troubleshooting
- `docs/IOS_QA_MATRIX.md`
  - device matrix
  - network transition tests
  - gateway restart tests
  - auth-expiry recovery tests
- `README.md`
  - “iOS Thin Client (MVP)” section with build/test commands
- `FEATURE_PARITY.md`
  - move iOS-related entries touched by this MVP from `🚫` to `🚧` or `✅` with notes.

**Step 4: Run verification to verify it passes**

Run:
```bash
bash scripts/check_ios_mvp.sh
cargo test --test ios_gateway_contract_integration -q
swift test --package-path apps/ios/IronClawClientKit
xcodegen generate --spec apps/ios/IronClawMobile/project.yml
xcodebuild -project apps/ios/IronClawMobile/IronClawMobile.xcodeproj -scheme IronClawMobile -destination 'platform=iOS Simulator,name=iPhone 16' test
```
Expected: PASS across checks/tests.

**Step 5: Commit**

```bash
git add docs/IOS_MVP_RUNBOOK.md docs/IOS_QA_MATRIX.md README.md FEATURE_PARITY.md scripts/check_ios_mvp.sh
git commit -m "docs: add iOS MVP runbook, QA matrix, and parity updates"
```

---

## Final verification gate (before PR)

Run:
```bash
cargo test --test ios_gateway_contract_integration -q
swift test --package-path apps/ios/IronClawClientKit
xcodegen generate --spec apps/ios/IronClawMobile/project.yml
xcodebuild -project apps/ios/IronClawMobile/IronClawMobile.xcodeproj -scheme IronClawMobile -destination 'platform=iOS Simulator,name=iPhone 16' test
```

Expected:
- All commands succeed with zero failing tests.
- iOS app can send chat message, stream response, handle approval, list/cancel jobs, and search/read memory against LAN and remote profile targets.

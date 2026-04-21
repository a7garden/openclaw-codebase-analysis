# Mobile & Desktop Apps Architecture

> *Part of the OpenClaw Architecture Analysis — see [01-overview.md](./01-overview.md)*

---

## 1. Overview

OpenClaw has native apps for iOS, Android, macOS, and a speech wake-word daemon called Swabble. All share a common protocol library (`OpenClawKit`) for gateway communication.

---

## 2. Shared Protocol: OpenClawKit

**Location:** `apps/shared/OpenClawKit/`

Swift package providing shared gateway protocol and UI components.

### 2.1 Structure

```
apps/shared/OpenClawKit/
├── Package.swift (62 lines)
├── OpenClawProtocol/
│   └── GatewayModels.swift          # Cross-platform wire types
├── OpenClawKit/
│   ├── GatewayChannel.swift         # WebSocket gateway client
│   ├── GatewayConnectChallengeSupport.swift
│   ├── GatewayTLSStore.swift
│   ├── BonjourServiceResolverSupport.swift
│   ├── CameraCommands.swift
│   ├── LocationCommands.swift
│   └── 60+ other utilities
└── OpenClawChatUI/
    ├── ChatComposer.swift
    ├── ChatView.swift
    ├── ChatViewModel.swift
    └── ChatMarkdownRenderer.swift
```

### 2.2 Gateway Protocol Version

```swift
let GATEWAY_PROTOCOL_VERSION = 3
```

---

## 3. iOS App

**Location:** `apps/ios/`
**Framework:** Swift 6 / SwiftUI
**Build:** Xcode via `project.yml` (XcodeGen/Tuplet)

### 3.1 Targets (from project.yml:355)

| Target | Platform | Description |
|--------|----------|-------------|
| `OpenClaw` | iOS 18+ | Main iOS app |
| `OpenClawShareExtension` | iOS | Share extension |
| `OpenClawActivityWidget` | iOS | Live Activities widget |
| `OpenClawWatchApp` | watchOS 11+ | watchOS companion |
| `OpenClawWatchExtension` | watchOS 11+ | watchOS extension |
| `OpenClawTests` | iOS | Unit tests |
| `OpenClawLogicTests` | iOS | Logic tests |

### 3.2 App Structure

**Entry point:** `apps/ios/Sources/OpenClawApp.swift:605`

```swift
@main
struct OpenClawApp: App {
  var body: some Scene {
    WindowGroup {
      OpenClawAppView()
    }
  }
}
```

### 3.3 App Delegate

**File:** `apps/ios/Sources/OpenClawApp.swift:24` (605 lines)

```swift
class OpenClawAppDelegate: NSObject, UIApplicationDelegate {
  func application(
    _ application: UIApplication,
    didFinishLaunchingWithOptions launchOptions: [UIApplication.LaunchOptionsKey: Any]? = nil
  ) -> Bool {
    // Push notification registration
    // Background task registration
    // Deep link handling
  }
}
```

### 3.4 NodeAppModel

**File:** `apps/ios/Sources/Model/NodeAppModel.swift:56` (200+ lines)

Observable state manager:

```swift
@Observable
class NodeAppModel {
  var nodeGateway: GatewayNodeSession    // Device capabilities
  var operatorGateway: GatewayNodeSession // Chat/talk/config
  var voiceWakeManager: VoiceWakeManager
  var talkModeManager: TalkModeManager

  // Service providers
  var cameraProvider: CameraService?
  var locationProvider: LocationService?
  var photosProvider: PhotosService?
  var contactsProvider: ContactsService?
  var calendarProvider: CalendarService?
  var motionProvider: MotionService?
}
```

### 3.5 Gateway Connection

**File:** `apps/ios/Sources/Gateway/GatewayConnectionController.swift`

Handles:
- Gateway discovery (Bonjour/DNS-SD)
- TLS trust evaluation
- Connection management

### 3.6 Voice Features

| Component | File | Description |
|----------|------|-------------|
| VoiceWakeManager | `Voice/VoiceWakeManager.swift` | Wake word detection |
| TalkModeManager | `Voice/TalkModeManager.swift` | Voice conversation |
| TalkOrbOverlay | `Voice/TalkOrbOverlay.swift` | Voice UI indicator |

### 3.7 Push Notifications

```
PushRegistrationManager.swift  — APNs registration
ExecApprovalNotificationBridge.swift — Approval request push handling
```

### 3.8 watchOS Support

- **WatchApp** — companion app on Apple Watch
- **WatchExtension** — runs on watchOS 11+
- **WatchMessagingService** — sends/receives messages via WatchConnectivity
- **WatchConnectivityTransport** — Watch <-> iPhone communication

---

## 4. Android App

**Location:** `apps/android/`
**Framework:** Kotlin / Jetpack Compose
**Build:** Gradle (build.gradle.kts, 291 lines)

### 4.1 Entry Points

```kotlin
// MainActivity.kt (ComponentActivity)
class MainActivity : ComponentActivity()

// NodeApp.kt (Application class)
class NodeApp : Application()

// NodeRuntime.kt (1308 lines)
class NodeRuntime(
  context: Context,
  prefs: SharedPreferences,
  canvas: CanvasController,
  camera: CameraCaptureManager,
  location: LocationCaptureManager,
  sms: SmsManager,
  discovery: GatewayDiscovery,
  operatorSession: GatewaySession,
  nodeSession: GatewaySession,
  chat: ChatController,
  talkMode: TalkModeManager,
  micCapture: MicCaptureManager,
  invokeDispatcher: InvokeDispatcher
)
```

### 4.2 Build Configuration

```kotlin
// build.gradle.kts
android {
  compileSdk = 36
  minSdk = 31
  targetSdk = 36

  ndk {
    abiFilters += listOf("armeabi-v7a", "arm64-v8a", "x86", "x86_64")
  }
}

productFlavors += listOf("play", "thirdParty")
// play: no SMS/call log permissions
// thirdParty: SMS/call log enabled
```

### 4.3 Dependencies

| Library | Version |
|---------|---------|
| Compose BOM | 2026.02.00 |
| CameraX | 1.5.2 |
| OkHttp | 5.3.2 |
| dnsjava | 3.6.4 |

### 4.4 Gateway Discovery

**File:** `apps/android/app/src/main/java/ai/openclaw/app/gateway/`

```kotlin
GatewayDiscovery.kt    // Bonjour + DNS-SD
GatewayEndpoint.kt     // Endpoint management
GatewayTls.kt         // TLS trust evaluation
GatewaySession.kt     // WebSocket client (500+ lines)
```

### 4.5 Node Invoke Dispatcher

**File:** `apps/android/app/src/main/java/ai/openclaw/app/node/InvokeDispatcher.kt`

Routes `node.invoke` commands to handlers:

```kotlin
sealed class InvokeRequest {
  data class CameraCapture(val params: CameraParams) : InvokeRequest()
  data class LocationGet(val params: LocationParams) : InvokeRequest()
  data class NotificationSend(val params: NotificationParams) : InvokeRequest()
  // ...
}

class InvokeDispatcher {
  fun dispatch(request: InvokeRequest): InvokeResult
}
```

### 4.6 Voice

| Component | Description |
|-----------|-------------|
| MicCaptureManager | Microphone capture for voice |
| TalkModeManager | Voice conversation |
| VoiceWakeManager | Wake word detection |

---

## 5. macOS App

**Location:** `apps/macos/`
**Framework:** Swift 6 / SwiftUI
**Build:** Swift Package Manager

### 5.1 Structure

```
apps/macos/Sources/OpenClaw/
├── AppState.swift
├── GatewayConnection.swift
├── CanvasWindowController.swift
├── ChannelsStore.swift
└── ConfigStore.swift
```

### 5.2 Dependencies

- MenuBarExtraAccess
- Swift Subprocess
- swift-log
- Sparkle (auto-update)
- Peekaboo (screen recording detection)
- MLXAudio (audio processing)

---

## 6. Swabble

**Location:** `Swabble/`
**Framework:** Swift 6.2
**Build:** Swift Package Manager

A macOS speech wake-word daemon that runs alongside the OpenClaw gateway.

### 6.1 Package Structure

```swift
// Package.swift (56 lines)
products += [
  .library(name: "Swabble", targets: ["Swabble"]),
  .library(name: "SwabbleKit", targets: ["SwabbleKit"]),  // Shared wake-gate
  .executable(name: "swabble", targets: ["swabble"])      // CLI
]

platforms = [.macOS("15.0"), .iOS("17.0")]
```

### 6.2 Products

| Product | Description |
|---------|-------------|
| `Swabble` | Core daemon |
| `SwabbleKit` | Shared wake-gate library (used by iOS/macOS apps) |
| `swabble` | CLI tool |

---

## 7. Cross-Platform Connection Architecture

```
Native App
├── GatewayChannel / GatewaySession (WebSocket)
│   ├── TLS fingerprint verification
│   ├── Device auth (device identity + role tokens)
│   └── Reconnect with exponential backoff
├── NodeAppModel / NodeRuntime (per-platform state)
│   ├── nodeSession — node.invoke commands (camera, location, etc.)
│   └── operatorSession — chat, talk, config
└── GatewayDiscovery (mDNS/Bonjour + DNS-SD)
```

### 7.1 Discovery

Both iOS and Android support **gateway discovery via Bonjour/mDNS** — apps can find a local gateway without knowing its IP address.

### 7.2 TLS Trust

Both iOS and Android implement **TLS certificate fingerprint verification** to prevent MITM attacks on local connections.

---

## 8. Key Files

| File | Lines | Purpose |
|------|-------|---------|
| `apps/ios/Sources/OpenClawApp.swift` | 605 | iOS app entry (@main) |
| `apps/ios/Sources/Model/NodeAppModel.swift` | 200+ | iOS state manager |
| `apps/ios/project.yml` | 355 | iOS XcodeGen config |
| `apps/android/app/src/main/java/ai/openclaw/app/NodeRuntime.kt` | 1308 | Android core runtime |
| `apps/android/app/build.gradle.kts` | 291 | Android Gradle config |
| `apps/shared/OpenClawKit/Package.swift` | 62 | OpenClawKit package |
| `apps/shared/OpenClawKit/OpenClawKit/GatewayChannel.swift` | 500+ | Swift gateway client |
| `Swabble/Package.swift` | 56 | Swabble package |
# HawkSight - FalconEye iOS Client

The iPhone client application for FalconEye's distributed spatial mapping system. HawkSight captures spatial data with ARKit and LiDAR, transmits it to the Aerie server, and renders real-time through-wall skeleton overlays.

## Overview

HawkSight is an iOS application that:

- **Captures camera frames** at 30 FPS using AVFoundation
- **Tracks device pose** using ARKit's world tracking with LiDAR
- **Connects to Aerie server** via WebSocket for real-time communication
- **Authenticates** using token-based authentication
- **Transmits frames and poses** to the server for processing
- **Receives pose broadcasts** with detected skeletons in world coordinates
- **Renders skeleton overlays** on the live camera feed for through-wall visualization

## Technology Stack

- **Language**: Swift
- **UI Framework**: SwiftUI
- **AR Framework**: ARKit (ARWorldTrackingConfiguration)
- **Camera**: AVFoundation (AVCaptureSession)
- **Networking**: URLSession WebSocket API
- **Authentication**: CryptoKit (HMAC signatures)
- **Platform**: iOS 16+
- **Target Devices**: iPhone 12 Pro or later (LiDAR required)

## System Requirements

### Hardware Requirements
- **iPhone 12 Pro or later** with LiDAR sensor
- Models supported:
  - iPhone 12 Pro / 12 Pro Max
  - iPhone 13 Pro / 13 Pro Max
  - iPhone 14 Pro / 14 Pro Max
  - iPhone 15 Pro / 15 Pro Max
  - iPhone 16 Pro / 16 Pro Max (or later)

### Software Requirements
- **iOS 16.0+**
- **Developer Mode** enabled (Settings → Privacy & Security → Developer Mode)
- **Xcode 15+** for building
- **Same WiFi network** as Aerie server

### Permissions Required
- Camera access
- Local network access
- Location (for WiFi positioning)

## Quick Start

### Prerequisites

1. **Aerie server must be running**:
   ```bash
   # On MacBook
   cd FalconEye/Aerie
   swift run
   ```

2. **Get Aerie server IP address**:
   ```bash
   # On MacBook
   ifconfig | grep "inet " | grep -v 127.0.0.1
   # Example output: inet 192.168.1.100
   ```

### Installation

```bash
# Open project in Xcode
cd FalconEye/HawkSight
open HawkSight.xcodeproj
```

### Configuration

Before building, update the server URL:

1. Open `HawkSight/Config.swift` (or wherever server URL is defined)
2. Update the server IP address:
   ```swift
   let serverURL = "ws://192.168.1.100:8000/ws/connect"
   // Replace 192.168.1.100 with your Aerie server's IP
   ```

### Building & Running

1. **Connect physical iPhone** via USB
2. **Select device** in Xcode's device picker (top-left)
3. **Trust development certificate** on iPhone if prompted
4. **Build and run**: Product → Run (⌘R)

**IMPORTANT**: Cannot use iOS Simulator - ARKit LiDAR features require physical hardware.

### First Launch

1. **Grant permissions** when prompted:
   - Camera access
   - Local network access
   - Location access (if requested)

2. **Connection status** should show:
   - "Connecting..." → "Authenticating..." → "Connected"

3. **Test capture**:
   - Wave phone around to see ARKit working
   - Connection status should show FPS counter

## Project Structure

```
HawkSight/
├── HawkSight.xcodeproj/      # Xcode project file
├── HawkSight/
│   ├── HawkSightApp.swift    # App entry point (@main)
│   ├── ContentView.swift     # Main view
│   ├── Models/               # Data models (planned)
│   │   ├── CameraFrame.swift
│   │   ├── DevicePose.swift
│   │   ├── Joint.swift
│   │   └── DetectedPose.swift
│   ├── Views/                # SwiftUI views (planned)
│   │   ├── ARCameraView.swift
│   │   ├── SkeletonOverlay.swift
│   │   └── ConnectionStatus.swift
│   ├── Networking/           # WebSocket + protocol (planned)
│   │   ├── WebSocketClient.swift
│   │   ├── MessageHandler.swift
│   │   └── AuthManager.swift
│   ├── AR/                   # ARKit integration (planned)
│   │   ├── ARSessionManager.swift
│   │   ├── PoseCapture.swift
│   │   └── FrameCapture.swift
│   ├── Rendering/            # Skeleton rendering (planned)
│   │   ├── SkeletonRenderer.swift
│   │   └── CoordinateProjection.swift
│   └── Assets.xcassets/      # Images and colors
└── HawkSightTests/           # Unit tests
```

## App Architecture

### Data Flow

```
ARKit Session → Pose Capture → Frame Capture
                                     ↓
                              Serialization
                                     ↓
                           WebSocket Transmit → Aerie
                                     ↑
                           WebSocket Receive ← Aerie
                                     ↓
                          Pose Message Parse
                                     ↓
                       World → Image Projection
                                     ↓
                          Skeleton Rendering
                                     ↓
                           Camera Feed Overlay
```

### Key Components

#### 1. ARSessionManager (Phase 2)
Manages ARKit session:
- Initializes `ARWorldTrackingConfiguration`
- Captures device pose at 60 FPS
- Extracts camera intrinsics (focal length, principal point)
- Provides current pose on demand

#### 2. FrameCapture (Phase 2)
Handles camera frame capture:
- Uses `AVCaptureSession` for front/rear camera
- Captures frames at ~30 FPS
- Converts frames to JPEG (quality: 0.6)
- Manages frame buffer to prevent memory issues

#### 3. WebSocketClient (Phase 1)
Manages server communication:
- Connects to Aerie WebSocket endpoint
- Handles authentication flow
- Maintains connection state
- Automatic reconnection on disconnect
- Sends camera frames + poses
- Receives pose broadcasts

#### 4. SkeletonRenderer (Phase 5)
Renders visualization overlay:
- Projects world-frame poses to image coordinates
- Draws joints as circles (6-8px, green)
- Draws limbs as lines (cyan)
- Overlays on live camera feed at 30 FPS

## Network Protocol

### WebSocket Connection

**Endpoint**: `ws://[aerie-ip]:8000/ws/connect/{deviceId}`

**Connection Flow**:
1. Generate unique deviceId (UUID)
2. Connect to WebSocket with deviceId in URL
3. Send `AuthRequest` with token
4. Await `AuthResponse`
5. On success, transmit frames and receive poses

### Message Types

All messages use JSON encoding with Codable Swift models.

#### Outgoing Messages (to Aerie)

**AuthRequest**
```swift
struct AuthRequest: Codable {
    let deviceId: String
    let token: String
    let deviceInfo: DeviceInfo
}

struct DeviceInfo: Codable {
    let model: String        // e.g., "iPhone 14 Pro"
    let osVersion: String    // e.g., "iOS 17.1"
    let hasLiDAR: Bool
}
```

**CameraFrame** (Phase 2)
```swift
struct CameraFrame: Codable {
    let frameId: String
    let timestamp: Double
    let imageData: String          // base64 JPEG
    let devicePose: [[Float]]      // 4x4 matrix
    let intrinsics: CameraIntrinsics
}

struct CameraIntrinsics: Codable {
    let fx: Float    // Focal length X
    let fy: Float    // Focal length Y
    let cx: Float    // Principal point X
    let cy: Float    // Principal point Y
    let width: Int
    let height: Int
}
```

#### Incoming Messages (from Aerie)

**AuthResponse**
```swift
struct AuthResponse: Codable {
    let success: Bool
    let sessionId: String?
    let error: String?
}
```

**PoseBroadcast** (Phase 5)
```swift
struct PoseBroadcast: Codable {
    let timestamp: Double
    let poses: [DetectedPose]
}

struct DetectedPose: Codable {
    let personId: String
    let joints: [Joint]
    let sourceDevice: String
    let confidence: Float
}

struct Joint: Codable {
    let type: JointType    // enum: head, leftShoulder, etc.
    let position: [Float]  // [x, y, z] in world frame
    let confidence: Float
}
```

## Authentication (Phase 1)

### Token Generation

```swift
import CryptoKit

func generateAuthToken(deviceId: String, sharedSecret: String) -> String {
    let timestamp = Date().timeIntervalSince1970
    let message = "\(deviceId):\(timestamp)"
    let key = SymmetricKey(data: sharedSecret.data(using: .utf8)!)
    let signature = HMAC<SHA256>.authenticationCode(
        for: message.data(using: .utf8)!,
        using: key
    )
    return Data(signature).base64EncodedString()
}
```

### Authentication Flow

1. Generate or retrieve deviceId (UUID, stored in UserDefaults)
2. Generate auth token using HMAC signature
3. Send `AuthRequest` via WebSocket
4. Wait for `AuthResponse`
5. On success: store sessionId, update UI to "Connected"
6. On failure: show error, allow retry

## ARKit Integration (Phase 2)

### ARSession Setup

```swift
import ARKit

class ARSessionManager: NSObject, ARSessionDelegate {
    let session = ARSession()

    func start() {
        let config = ARWorldTrackingConfiguration()
        config.frameSemantics = [.sceneDepth]  // Enable LiDAR
        session.run(config)
        session.delegate = self
    }

    func session(_ session: ARSession, didUpdate frame: ARFrame) {
        // Extract device pose (4x4 transform matrix)
        let transform = frame.camera.transform

        // Extract camera intrinsics
        let intrinsics = frame.camera.intrinsics

        // Capture and transmit frame
    }
}
```

### Pose & Intrinsics Extraction

```swift
func extractDevicePose(from frame: ARFrame) -> simd_float4x4 {
    return frame.camera.transform
}

func extractIntrinsics(from frame: ARFrame) -> CameraIntrinsics {
    let intrinsics = frame.camera.intrinsics
    let resolution = frame.camera.imageResolution

    return CameraIntrinsics(
        fx: intrinsics[0, 0],
        fy: intrinsics[1, 1],
        cx: intrinsics[2, 0],
        cy: intrinsics[2, 1],
        width: Int(resolution.width),
        height: Int(resolution.height)
    )
}
```

## Frame Transmission (Phase 2)

### Rate Control

```swift
class FrameTransmitter {
    let targetFPS: Double = 30
    private var lastTransmitTime: Date?

    func shouldTransmit() -> Bool {
        guard let lastTime = lastTransmitTime else { return true }
        let elapsed = Date().timeIntervalSince(lastTime)
        return elapsed >= (1.0 / targetFPS)
    }

    func transmit(_ frame: CameraFrame) async {
        if shouldTransmit() {
            await webSocket.send(frame)
            lastTransmitTime = Date()
        }
    }
}
```

### Frame Buffering

```swift
class FrameBuffer {
    private var queue: [CameraFrame] = []
    private let maxQueueSize = 30  // 1 second at 30 FPS

    func enqueue(_ frame: CameraFrame) {
        queue.append(frame)
        if queue.count > maxQueueSize {
            queue.removeFirst()
        }
    }
}
```

## Pose Rendering (Phase 5)

### World → Image Projection

```swift
func projectWorldToImage(
    worldPoint: simd_float3,
    devicePose: simd_float4x4,
    intrinsics: CameraIntrinsics
) -> CGPoint? {
    // World → Device
    let devicePoseInverse = devicePose.inverse
    let devicePoint = devicePoseInverse * simd_float4(worldPoint, 1)

    // Check if in front of camera
    guard devicePoint.z > 0 else { return nil }

    // Camera 3D → Image 2D (perspective projection)
    let x = (intrinsics.fx * devicePoint.x / devicePoint.z) + intrinsics.cx
    let y = (intrinsics.fy * devicePoint.y / devicePoint.z) + intrinsics.cy

    // Verify within bounds
    guard x >= 0 && x < Float(intrinsics.width) &&
          y >= 0 && y < Float(intrinsics.height) else {
        return nil
    }

    return CGPoint(x: CGFloat(x), y: CGFloat(y))
}
```

### Skeleton Overlay

```swift
import SwiftUI

struct SkeletonOverlay: View {
    let poses: [DetectedPose]
    let devicePose: simd_float4x4
    let intrinsics: CameraIntrinsics

    var body: some View {
        Canvas { context, size in
            for pose in poses {
                drawSkeleton(pose, in: context)
            }
        }
    }

    func drawSkeleton(_ pose: DetectedPose, in context: GraphicsContext) {
        // Project joints to image coordinates
        let projectedJoints = pose.joints.compactMap { joint -> (JointType, CGPoint)? in
            let worldPos = simd_float3(
                joint.position[0],
                joint.position[1],
                joint.position[2]
            )
            guard let imagePos = projectWorldToImage(
                worldPoint: worldPos,
                devicePose: devicePose,
                intrinsics: intrinsics
            ) else { return nil }
            return (joint.type, imagePos)
        }

        // Draw joints (green circles)
        for (_, point) in projectedJoints {
            let circle = Circle()
                .path(in: CGRect(x: point.x - 4, y: point.y - 4, width: 8, height: 8))
            context.stroke(circle, with: .color(.green), lineWidth: 2)
        }

        // Draw limbs (cyan lines)
        drawLimbs(projectedJoints, in: context)
    }
}
```

## UI Components

### Connection Status Display

```swift
struct ConnectionStatusView: View {
    let state: ConnectionState
    let frameRate: Double
    let latency: TimeInterval?

    var body: some View {
        VStack(alignment: .leading) {
            HStack {
                Circle()
                    .fill(statusColor)
                    .frame(width: 10, height: 10)
                Text(statusText)
                    .font(.caption)
            }

            if let latency = latency {
                Text("Latency: \(Int(latency * 1000))ms")
                    .font(.caption2)
            }

            Text("FPS: \(Int(frameRate))")
                .font(.caption2)
        }
        .padding(8)
        .background(.ultraThinMaterial)
        .cornerRadius(8)
    }
}
```

### Main Camera View

```swift
struct ARCameraView: View {
    @State private var arManager = ARSessionManager()
    @State private var connectionManager = ConnectionManager()
    @State private var receivedPoses: [DetectedPose] = []

    var body: some View {
        ZStack {
            // AR camera feed (full screen)
            ARViewRepresentable(session: arManager.session)
                .ignoresSafeArea()

            // Skeleton overlay
            SkeletonOverlay(
                poses: receivedPoses,
                devicePose: arManager.currentPose,
                intrinsics: arManager.currentIntrinsics
            )

            // Connection status (top-right)
            VStack {
                HStack {
                    Spacer()
                    ConnectionStatusView(
                        state: connectionManager.state,
                        frameRate: arManager.frameRate,
                        latency: connectionManager.latency
                    )
                }
                Spacer()
            }
            .padding()
        }
        .onAppear {
            arManager.start()
            connectionManager.connect(to: serverURL)
        }
    }
}
```

## Testing

### Unit Tests

Run in Xcode: Product → Test (⌘U)

Key tests:
- Coordinate projection (world → image)
- Message encoding/decoding
- Frame buffer management
- Authentication token generation

### Integration Testing

1. **AR Session**: Verify ARKit initialization
2. **WebSocket**: Connect to test server
3. **Auth Flow**: Send auth, verify response
4. **Frame Transmission**: Send frames, verify Aerie receives
5. **Pose Reception**: Receive mock poses, verify rendering

### Manual Testing

1. Run Aerie server on MacBook
2. Update serverURL in HawkSight
3. Build and run on physical iPhone
4. Verify connection status: "Connected"
5. Wave phone around, check pose transmission
6. Have person in view, verify skeleton renders

## Performance Considerations

### Memory Management
- Limit pose history to 30 frames (1 second)
- Release old camera frames immediately
- Use weak references for delegates
- Monitor with Instruments (Xcode → Product → Profile)

### Frame Rate Targets
- ARKit poses: 60 FPS capture
- Frame transmission: 30 FPS
- Overlay rendering: 30 FPS (camera frame rate)

### Network Optimization
- JPEG quality: 0.6 (balance size/quality)
- Target frame size: <500KB
- Future: H.264 streaming (Phase 6)

## Common Issues

### ARKit not initializing

**Symptoms**: Black screen, no camera feed

**Solutions**:
- Verify device has LiDAR (Settings → General → About → Model)
- Check camera permissions (Settings → HawkSight → Camera)
- Ensure not running in Simulator
- Enable Developer Mode (Settings → Privacy & Security)

### WebSocket connection fails

**Symptoms**: Status shows "Connecting..." indefinitely

**Solutions**:
- Verify Aerie is running: `curl http://[server-ip]:8000`
- Check both devices on same WiFi
- Verify serverURL uses correct IP (not localhost)
- Check MacBook firewall settings
- Restart WiFi on both devices

### Skeleton not rendering

**Symptoms**: Connected but no overlay visible

**Solutions**:
- Check pose broadcast messages received (add logging)
- Verify coordinate projection math
- Ensure device pose is being captured
- Log projected points to debug
- Check intrinsics are valid (not NaN/zero)

### App crashes on launch

**Solutions**:
- Check crash logs: Xcode → Window → Devices and Simulators
- Verify ARKit configuration
- Check for force-unwrapping nil values
- Monitor memory usage (should be <500MB)

### Poor performance / lag

**Solutions**:
- Reduce JPEG quality (default: 0.6)
- Lower frame transmission rate (default: 30 FPS)
- Check network latency
- Profile with Instruments

## Future Enhancements

### Phase 6: Performance Optimization
- H.264 video streaming
- Adaptive frame rate based on motion
- Predictive rendering for smoother display
- Core Animation for joint animations

### Phase 7: Multi-Person Tracking
- Different colors per person
- Person ID labels on overlay
- Show which device detected each person

### Phase 8: LiDAR Mesh Integration
- Extract and transmit spatial mesh
- Render semi-transparent walls
- Proper occlusion of skeleton joints behind walls

## Privacy & Security

- Camera/ARKit permissions required
- No local storage of camera frames
- Secure WebSocket (WSS) recommended for production
- Token-based authentication prevents unauthorized access
- Privacy-focused: pose detection only (no face detection)
- All data transmitted to Aerie for processing

## References

- [ARKit Documentation](https://developer.apple.com/arkit/)
- [SwiftUI](https://developer.apple.com/xcode/swiftui/)
- [WebSocket API](https://developer.apple.com/documentation/foundation/urlsessionwebsockettask)
- [CryptoKit](https://developer.apple.com/documentation/cryptokit)

## Support

For component-specific issues:
- iOS client bugs: Open issue in HawkSight repository
- Server bugs: Check Aerie repository
- System integration: Open issue in FalconEye repository

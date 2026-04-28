# 03 — iOS Swift Components

🔑 `LiveKitComponents` is the SwiftUI counterpart to the React lib — `RoomScope` owns the connection, environment objects expose room/participant state, declarative views render tracks.

Source: https://docs.livekit.io/reference/components/swift/documentation/livekitcomponents (404 at fetch time; canonical landing https://docs.livekit.io/reference/components/swift/ also 404'd — see https://github.com/livekit/components-swift for the package and DocC source)

## Install (SwiftPM)

```swift
// Package.swift
.package(url: "https://github.com/livekit/components-swift", from: "0.1.0"),
.package(url: "https://github.com/livekit/client-sdk-swift", from: "2.0.0"),
```

Targets depend on `LiveKitComponents` and `LiveKit`.

## Core Building Blocks

- `RoomScope` — root view; takes `url`, `token`, `connect`, `enableMicrophone`, `enableCamera`. Injects `Room` into environment.
- `ForEachParticipant` — iterates `Room.allParticipants`, providing `@EnvironmentObject Participant`.
- `ForEachTrack(filter:)` — iterates published tracks for the current participant scope; filter by `.source(.camera)` / `.source(.microphone)` / `.kind(.video)`.
- `ParticipantView` — default tile.
- `VideoTrackView` / `AudioVisualizer` — render a `TrackPublication`.
- `BarAudioVisualizer(audioTrack:barCount:)` — agent-style waveform.

## Environment Objects

- `@EnvironmentObject var room: Room`
- `@EnvironmentObject var participant: Participant` (inside `ForEachParticipant`)
- `@EnvironmentObject var track: TrackPublication` (inside `ForEachTrack`)

## Connection State

Use `room.connectionState` (`.disconnected | .connecting | .connected | .reconnecting`) to drive UI.

## Voice Assistant View

```swift
import SwiftUI
import LiveKit
import LiveKitComponents

struct AgentView: View {
    let url: String
    let token: String

    var body: some View {
        RoomScope(url: url, token: token, connect: true, enableMicrophone: true) {
            VStack {
                ForEachParticipant(filter: .kind(.agent)) {
                    ForEachTrack(filter: .source(.microphone)) {
                        BarAudioVisualizer(barCount: 5)
                    }
                }
                DisconnectButton()
            }
        }
    }
}
```

## Notes

- Add `NSMicrophoneUsageDescription` (and camera if used) to `Info.plist`.
- For background audio, enable the Audio background mode capability.
- `room.localParticipant.setMicrophone(enabled:)` / `setCamera(enabled:)` for control.

## Tags
[[LiveKit]] [[Swift]] [[iOS]] [[Frontends]]

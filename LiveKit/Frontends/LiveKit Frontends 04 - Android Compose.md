# 04 — Android Compose Components

🔑 `livekit-android-compose-components` mirrors the React API in Kotlin — `RoomScope` connects, scoped composables expose participant/track state via `CompositionLocal`.

Source: https://docs.livekit.io/reference/components/android/

## Install

`settings.gradle.kts`:

```kotlin
dependencyResolutionManagement {
    repositories {
        google()
        mavenCentral()
        maven { url = URI("https://jitpack.io") }
    }
}
```

`build.gradle.kts` (app):

```kotlin
dependencies {
    implementation("io.livekit:livekit-android-compose-components:<latest>")
    implementation("io.livekit:livekit-android:<latest>")
}
```

Latest version: https://github.com/livekit/components-android/releases

## Core Composables

- `RoomScope(url, token, audio, video, connect, content)` — connects and provides `Room` via `LocalRoom`.
- `ParticipantScope(participant) { ... }` — scopes a sub-tree to one participant.
- `TrackReferences()` / `rememberTracks(sources = listOf(Source.MICROPHONE, Source.CAMERA))` — list current track refs.
- `VideoTrack(videoTrack: VideoTrack, modifier: Modifier)` — render video.
- `AudioBarVisualizer(audioTrack:, barCount:)` — agent waveform.

## Scopes

Composables read room state through:
- `LocalRoom.current` → `Room`
- `LocalParticipant.current` → `Participant`
- `LocalTrackReference.current` → `TrackReference`

`rememberParticipants()`, `rememberLocalParticipant()`, `rememberRemoteParticipants()` are remember-style helpers.

## Permissions

Request `RECORD_AUDIO` (and `CAMERA` / `BLUETOOTH_CONNECT` if used) at runtime via `rememberLauncherForActivityResult` before calling `RoomScope`.

## Voice Agent Screen

```kotlin
@Composable
fun AgentScreen(url: String, token: String) {
    RoomScope(
        url = url,
        token = token,
        audio = true,
        video = false,
        connect = true,
    ) {
        val agentTracks = rememberTracks(sources = listOf(Track.Source.MICROPHONE))
            .filter { it.participant.kind == Participant.Kind.AGENT }

        Column(Modifier.fillMaxSize(), Arrangement.Center, Alignment.CenterHorizontally) {
            agentTracks.firstOrNull()?.let { ref ->
                AudioBarVisualizer(audioTrack = ref.publication.track as AudioTrack, barCount = 5)
            }
            DisconnectButton()
        }
    }
}
```

## Notes

- Foreground service required for background mic capture (LiveKit ships `AudioForegroundService`).
- Use `Room.e2eeOptions` for end-to-end encryption.
- Compose recompositions are wired through `Room` `Flow`s — no manual `LaunchedEffect` plumbing needed for participant lists.

## Tags
[[LiveKit]] [[Android]] [[Compose]] [[Frontends]]

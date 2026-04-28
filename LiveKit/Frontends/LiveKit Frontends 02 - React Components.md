# 02 — React Components

🔑 `@livekit/components-react` is a hooks-and-context library where `LiveKitRoom` owns the connection and 80+ hooks expose every slice of room state.

Source: https://docs.livekit.io/reference/components/react/

## Install

```bash
pnpm add @livekit/components-react @livekit/components-styles livekit-client
```

```tsx
import "@livekit/components-styles";
```

## Featured Components

- `LiveKitRoom` — root provider; takes `serverUrl`, `token`, `connect`, `audio`, `video`, `onDisconnected`.
- `RoomAudioRenderer` — mounts every remote audio track to a hidden `<audio>` element. Required for sound.
- `VideoTrack` / `AudioTrack` — render a single `TrackReference`.
- `ParticipantTile` — full participant card with name, mute indicator, video.

## Layout Components

- `GridLayout` — auto-grid of tiles.
- `FocusLayout` — one big tile + filmstrip.
- `CarouselLayout` — horizontally scrolling overflow.

## Prefabs (production-ready)

- `<VideoConference />` — entire meeting UI.
- `<AudioConference />` — voice-only.
- `<Chat />` — data-channel chat.
- `<ControlBar />` — mic, cam, screen-share, leave.
- `<MediaDeviceMenu />` — device picker.
- `<PreJoin onSubmit={...} />` — name + device check.

## Core Hooks

- `useTracks([Track.Source.Camera, Track.Source.Microphone])` — reactive `TrackReference[]`.
- `useParticipants()`, `useLocalParticipant()`, `useRemoteParticipants()`.
- `useConnectionState()`, `useRoomContext()`, `useRoomInfo()`.
- `useDataChannel(topic, onMessage)` — typed pub/sub.
- `useChat()` — `{ chatMessages, send, isSending }`.
- `useVoiceAssistant()` — agent-aware: `{ state, agent, audioTrack, agentTranscriptions }`.
- `useTrackTranscription(trackRef)` — stream STT segments.

## Minimal Voice Agent Frontend

```tsx
import {
  LiveKitRoom,
  RoomAudioRenderer,
  useVoiceAssistant,
  BarVisualizer,
} from "@livekit/components-react";

function Assistant() {
  const { state, audioTrack } = useVoiceAssistant();
  return (
    <div>
      <p>state: {state}</p>
      <BarVisualizer state={state} barCount={5} trackRef={audioTrack} />
    </div>
  );
}

export function App({ token, url }: { token: string; url: string }) {
  return (
    <LiveKitRoom serverUrl={url} token={token} connect audio>
      <Assistant />
      <RoomAudioRenderer />
    </LiveKitRoom>
  );
}
```

## Tags
[[LiveKit]] [[React]] [[Frontends]]

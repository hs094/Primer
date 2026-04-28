# 08 — Vision (Images and Video)

🔑 Push images into chat context, sample frames from a published video track, or stream live video into a realtime model — and respond with byte streams or virtual avatar video.

Source: https://docs.livekit.io/agents/build/vision/

## Inputs
- **Static images** — added to the agent's chat context (e.g. screenshot from frontend, doc page)
- **Sampled video frames** — pull a frame from a participant's video track on demand
- **Live video** — stream the full video track into a vision-capable realtime model (e.g. Gemini Realtime)

## Add an image to chat context
```python
from livekit.agents import ChatContext, ChatMessage, ChatImage

ctx = ChatContext()
ctx.append(
    ChatMessage(
        role="user",
        content=[
            "What do you see?",
            ChatImage(image=image_bytes),  # or a remote URL
        ],
    )
)
```

## Sample a frame from a video track
```python
from livekit import rtc

async def grab_frame(track: rtc.RemoteVideoTrack) -> rtc.VideoFrame:
    stream = rtc.VideoStream(track)
    async for event in stream:
        return event.frame
```

## Live video to a realtime model
```python
from livekit.plugins import google

session = AgentSession(
    llm=google.beta.realtime.RealtimeModel(model="gemini-2.0-flash-exp"),
)
# subscribe to the user's video track; the plugin forwards frames to the model
```

## Outputs
- **Byte streams** — send images/files back to the frontend
- **Virtual avatars** — drive a video avatar synchronized with TTS

## Reference recipe
Gemini Vision Assistant — full multimodal example showing live video + voice in, voice out.

## Tags
[[LiveKit]] [[Agents]] [[Vision]] [[Video]] [[Multimodal]] [[Gemini]]

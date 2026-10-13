# VoiceOne

**VoiceOne** is an open-source runtime and framework for building real-time voice AI agents. It combines a **Pipecat-style frame pipeline** (composable processors, streaming STT → LLM → TTS) with **LiveKit-style orchestration** (agent servers, job dispatch, isolated sessions, and WebRTC transport).

Use it when you need low-latency conversational agents that work on phone lines, browsers, or custom transports—with provider-agnostic plugins and production-oriented lifecycle management.

---

## Why VoiceOne

Voice conversations break when you bolt APIs together by hand: turn-taking, barge-in, partial transcripts, token streaming, and session cleanup all fight each other. VoiceOne treats voice AI as **infrastructure**:

| Concern | How VoiceOne handles it |
|--------|-------------------------|
| **Streaming** | Frames flow through a pipeline; LLM tokens can reach TTS before the full reply is generated |
| **Realtime media** | Transport adapters (WebRTC, SIP, WebSocket) normalize audio in/out |
| **Scale & isolation** | Agent server accepts dispatches; each session runs in its own worker/process |
| **Interruptions** | System frames (VAD, cancel, end) bypass normal queues with priority |
| **Observability** | Session traces, metrics hooks, and transcript aggregation built into the runtime |

Inspired by [Pipecat](https://github.com/pipecat-ai/pipecat) (pipeline + frames) and [LiveKit Agents](https://github.com/livekit/agents) (server lifecycle + realtime rooms)—without locking you to a single cloud or vendor.

---

## Architecture

```text
                    ┌─────────────────────────────────────┐
                    │           Agent Server              │
                    │  register · dispatch · drain · LB   │
                    └─────────────────┬───────────────────┘
                                      │ Job
                                      ▼
                    ┌─────────────────────────────────────┐
                    │         Session / Worker            │
                    │   (one voice agent per session)     │
                    └─────────────────┬───────────────────┘
                                      │
        ┌─────────────────────────────┴─────────────────────────────┐
        │                      Pipeline                              │
        │  transport.in → VAD → STT → context → LLM → TTS → out    │
        └─────────────────────────────────────────────────────────────┘
                                      │
                    optional: tools · RAG · guardrails · metrics
```

### Core concepts

**Frames**  
Typed packets that move through the system: audio chunks, transcriptions, LLM text deltas, TTS audio, tool calls, and control signals (start/stop/cancel). Most data flows **downstream**; control and metrics can flow **upstream**.

**Processors**  
Small, async units that consume and emit frames. Examples: STT, context aggregator, LLM, TTS, turn detector, interruption handler. Processors preserve ordering within their lane and pass through frames they do not handle.

**Pipeline**  
An ordered graph of processors (linear, parallel branches, or gated sub-pipelines). The pipeline runner schedules work so streaming stages overlap in time.

**Transport**  
Bridges the outside world to frames: microphone/speaker tracks, WebRTC rooms, telephony, or file/stream sources for development.

**Worker**  
Owns one pipeline instance for one conversation. A **WorkerRunner** manages multiple workers and optional cross-worker messaging (e.g. handoff or supervisor agents).

**Agent server**  
Long-lived process that registers capacity, receives **dispatch** requests, spawns a **job** (session entrypoint), and drains gracefully on deploy or shutdown.

**Session**  
The user-facing conversation boundary: participants, conversation state, tool registry, and shutdown semantics (drain speech, flush transcripts, disconnect).

---

## Pipeline modes

VoiceOne supports the same architectural families common in production voice stacks:

1. **Cascaded (STT → LLM → TTS)** — Maximum control; swap STT, model, or voice independently. Best when you need tools, RAG, or strict text-in-the-middle.
2. **Realtime (speech-to-speech)** — Single model path for lowest end-to-end latency; use when expressiveness and speed dominate.
3. **Half-cascade** — Realtime understanding + separate TTS for branded or controllable output voice.

Configure mode per session or per agent definition.

---

## Quick start

> **Status:** VoiceOne is under active development. APIs below describe the intended developer experience; pin to tagged releases once published.

### Prerequisites

- Python 3.11+ (primary SDK) or Node.js 20+ (optional SDK)
- API keys for your chosen STT / LLM / TTS providers (or self-hosted equivalents)
- For WebRTC: a compatible SFU or VoiceOne’s bundled dev transport

### Install

```bash
pip install voiceone
# optional extras
pip install "voiceone[webrtc,openai,deepgram,cartesia]"
```

### Minimal voice agent

```python
from voiceone import AgentServer, Pipeline, Transport
from voiceone.processors import STT, LLM, TTS, ContextAggregator
from voiceone.plugins import openai, deepgram, cartesia

server = AgentServer()

@server.session()
async def entrypoint(ctx):
    transport = Transport.webrtc(room=ctx.room)

    pipeline = Pipeline([
        transport.input(),
        STT(deepgram.STT(model="nova-3")),
        ContextAggregator.user(),
        LLM(openai.LLM(model="gpt-4.1-mini"), system="You are a helpful voice assistant."),
        TTS(cartesia.TTS(voice="…")),
        transport.output(),
        ContextAggregator.assistant(),
    ])

    await pipeline.run(ctx)

if __name__ == "__main__":
    server.start()
```

### Run locally

```bash
export OPENAI_API_KEY=...
export DEEPGRAM_API_KEY=...
python agent.py dev
```

Connect from the included web playground or your own client using the transport adapter documented for your deployment target.

---

## Plugins & providers

VoiceOne uses a **plugin interface** so pipelines stay stable when you change vendors:

- **STT** — streaming speech recognition with partial and final transcripts  
- **LLM** — chat completions with streaming tokens and tool calling  
- **TTS** — streaming synthesis with configurable voice and format  
- **VAD / turn detection** — end-of-utterance and barge-in signals  
- **Realtime** — unified speech models where supported  

Community and first-party plugins follow the same constructor + frame contract; see `docs/plugins.md` (coming soon).

---

## Production features

- **Dispatch** — explicit job APIs or automatic join when a user enters a room/channel  
- **Process isolation** — one bad session does not take down the server  
- **Graceful drain** — finish in-flight speech and close I/O on deploy  
- **Load awareness** — capacity hints for schedulers and autoscalers  
- **Telephony** — SIP/PSTN adapters as transport plugins  
- **Hooks** — metrics, logging, redaction, and custom guardrails as processors  

---

## Project layout (planned)

```text
voiceone/
├── voiceone/           # Core runtime: frames, pipeline, worker, server
├── plugins/            # Provider adapters
├── transports/         # WebRTC, websocket, telephony
├── examples/           # Reference agents
├── docs/               # Guides and API reference
└── tests/
```

---

## Roadmap

- [ ] Frame types and processor base class  
- [ ] Pipeline runner with system-frame priority  
- [ ] WebRTC transport and dev playground  
- [ ] Agent server + job dispatch  
- [ ] Reference STT / LLM / TTS plugins  
- [ ] Turn detection and interruption handling  
- [ ] Observability: transcripts, traces, latency breakdown  
- [ ] Node.js SDK parity  

---

## Contributing

Contributions are welcome: processors, transports, plugins, docs, and examples. Please open an issue to discuss larger design changes so frame contracts and session semantics stay coherent.

```bash
git clone https://github.com/your-org/voiceone.git
cd voiceone
pip install -e ".[dev]"
pytest
```

---

## License

Apache-2.0 — see [LICENSE](LICENSE) (to be added).

---

## Acknowledgments

VoiceOne’s design learns from the excellent work in the **[Pipecat](https://github.com/pipecat-ai/pipecat)** and **[LiveKit Agents](https://github.com/livekit/agents)** ecosystems. This project is independent and not affiliated with LiveKit or Pipecat.

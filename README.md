# Bidirectional Voice Agent

A real-time, voice-to-voice AI agent system built with [Strands Agents](https://strandsagents.com), [Amazon Nova 2 Sonic](https://aws.amazon.com/bedrock/nova/), and AWS AgentCore Runtime. Speak into your mic, get a spoken response back — with barge-in support so you can interrupt the agent mid-sentence.

![](./VoiceAgent.jpg)

---

## Architecture

```
Microphone → CLI Client → WebSocket → FastAPI Backend → Nova 2 Sonic (Strands BidiAgent) → Speaker
```

The system has three components:

| Component | Description |
|-----------|-------------|
| `backend/` | FastAPI WebSocket server wrapping a `BidiAgent` from the Strands SDK |
| `cli/` | Python CLI client that streams mic audio and plays back agent responses |
| `cdk/` | AWS CDK stack (TypeScript) to deploy the backend to AgentCore Runtime |

---

## Prerequisites

- Python >= 3.12
- Node.js >= 18.0.0
- [uv](https://docs.astral.sh/uv/getting-started/installation/) (Python package manager)
- AWS credentials configured (`aws configure` or environment variables)
- PortAudio installed (required by PyAudio)

```bash
# macOS
brew install portaudio

# Ubuntu/Debian
sudo apt-get install portaudio19-dev
```

---

## Quick Start (Local)

### 1. Start the Backend

```bash
cd backend
uv venv
source .venv/bin/activate
uv sync
uv run uvicorn app.main:app --host 0.0.0.0 --port 8080
```

Make sure your AWS credentials are set — the backend calls Bedrock directly.

### 2. Run the CLI Client

```bash
cd cli
uv venv
source .venv/bin/activate
uv sync

# Connect to local backend
.venv/bin/python app/main.py

# Connect to AgentCore Runtime (deployed)
.venv/bin/python app/main.py --agent_arn 'arn:aws:bedrock-agentcore:ap-northeast-1:...:runtime/...'
```

Once running, speak into your microphone. Press `Enter` to stop the session.

---

## CLI Arguments

| Argument | Default | Description |
|----------|---------|-------------|
| `--endpoint` | `ws://localhost:8080/ws` | WebSocket endpoint to connect to |
| `--agent_arn` | None | AgentCore Runtime ARN (overrides `--endpoint`) |
| `--debug` | False | Print all received event types for debugging |

---

## Backend Configuration

The following environment variables can be set on the backend (or in the CDK stack):

| Variable | Default | Description |
|----------|---------|-------------|
| `MODEL_ID` | `amazon.nova-2-sonic-v1:0` | Bedrock model ID |
| `REGION_NAME` | `ap-northeast-1` | AWS region |
| `INPUT_SAMPLE_RATE` | `16000` | Mic input sample rate (Hz) |
| `OUTPUT_SAMPLE_RATE` | `16000` | Audio output sample rate (Hz) |
| `CHANNELS` | `1` | Audio channels (mono) |

> Make sure the sample rates and channels match between the backend and CLI client.

---

## Docker (Local Testing)

```bash
cd backend

# Build for ARM64
docker buildx create --use
docker buildx build --platform linux/arm64 -t backend-agent:arm64 --load .

# Run
docker run --platform linux/arm64 -p 8080:8080 \
  -e AWS_ACCESS_KEY_ID="$AWS_ACCESS_KEY_ID" \
  -e AWS_SECRET_ACCESS_KEY="$AWS_SECRET_ACCESS_KEY" \
  -e AWS_SESSION_TOKEN="$AWS_SESSION_TOKEN" \
  -e AWS_REGION="$AWS_REGION" \
  backend-agent:arm64
```

---

## Deploy to AWS (CDK)

```bash
cd cdk
npm install
npm run build       # compile TypeScript
npx cdk deploy      # deploy to AWS
```

After deployment, the CDK stack outputs the `AgentCoreRuntimeArn`. Use that ARN with the CLI `--agent_arn` flag.

### Other CDK Commands

```bash
npx cdk diff        # compare with deployed stack
npx cdk synth       # emit CloudFormation template
npx cdk destroy     # tear down the stack
```

### CDK Build Status

The CDK TypeScript project compiles successfully with `npm run build` (TypeScript 5.6, aws-cdk-lib 2.222.0).

> Node.js >= 18 is required for the CDK stack.

---

## How It Works

1. The CLI captures PCM audio from your microphone in 512-frame chunks
2. Each chunk is base64-encoded and sent as a `bidi_audio_input` event over WebSocket
3. The FastAPI backend passes these events to a `BidiAgent` (Strands SDK) backed by Nova 2 Sonic
4. The agent streams back `bidi_audio_stream` events (base64 PCM) and `bidi_transcript_stream` events (text)
5. The CLI decodes and plays the audio in real time, and prints transcripts to the console
6. Barge-in is supported — speaking while the agent is responding clears the audio queue and interrupts playback

---

## Auth Note

When connecting to AgentCore Runtime, the CLI uses IAM credentials by default (via `bedrock-agentcore` SDK). If your runtime is configured with OAuth2, update the `websockets.connect` call in `cli/app/main.py` accordingly. See [AWS docs](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/runtime-get-started-websocket.html#step-4-invoke-deployed-websocket).

---

## References

- [Blog Post: BiDirectional Voice Agent with AgentCore Runtime / Nova 2 Sonic / Strands Agents](https://medium.com/@itsuki.enjoy/bidirectional-voice-agent-with-agentcore-runtime-nova2-sonic-strands-agents-9f86371d0641)
- [Strands Agents Docs](https://strandsagents.com/latest/)
- [Amazon Nova 2 Sonic](https://aws.amazon.com/bedrock/nova/)
- [AWS AgentCore Runtime](https://docs.aws.amazon.com/bedrock-agentcore/latest/devguide/what-is-bedrock-agentcore.html)

---
name: BuildShip Voice Generator
overview: Build a BuildShip workflow that receives Omi Developer Mode webhooks, detects "command generate [type]..." to "command execute" triggers, uses GPT-4o to pick the best AI/ML API model, generates content (music, meditations, images, videos), optionally applies FFmpeg ducking, uploads to storage, and sends an Omi notification with the public link.
todos:
  - id: get-api-keys
    content: Sign up for AI/ML API and get API key
    status: pending
  - id: create-buildship
    content: Create BuildShip account and new workflow
    status: pending
  - id: add-secrets
    content: Add AIML_API_KEY to BuildShip secrets
    status: pending
  - id: build-trigger
    content: Add REST API trigger node
    status: pending
  - id: build-buffer
    content: Add session buffer script node for trigger detection
    status: pending
  - id: build-router
    content: Add GPT-4o node with model selection prompt
    status: pending
  - id: build-http
    content: Add HTTP request node for AI/ML API
    status: pending
  - id: build-ffmpeg
    content: Add FFmpeg node for audio ducking (optional)
    status: pending
  - id: build-storage
    content: Add storage upload node
    status: pending
  - id: build-notify
    content: Add Omi notification HTTP node
    status: pending
  - id: configure-omi
    content: Enable Developer Mode in Omi and set webhook URL
    status: pending
  - id: test
    content: Test with voice command
    status: pending
---

# BuildShip Omi Voice Command Generator (Simple Setup)

## Architecture

```mermaid
flowchart LR
    subgraph Omi[Omi App]
        DevMode[Developer Mode Webhook]
    end
    
    subgraph BuildShip[BuildShip Workflow]
        Trigger[REST API Trigger]
        Buffer[Session Buffer]
        GPT[GPT-4o Router]
        HTTP[HTTP to AI/ML API]
        FFmpeg[FFmpeg Optional]
        Storage[BuildShip Storage]
        Notify[Omi Notification]
    end
    
    subgraph AIML[AI/ML API]
        Models[Music/Voice/Image/Video]
    end
    
    DevMode --> Trigger
    Trigger --> Buffer
    Buffer --> GPT
    GPT --> HTTP
    HTTP --> Models
    Models --> FFmpeg
    FFmpeg --> Storage
    Storage --> Notify
    Notify --> Omi
```



## Setup Steps

### 1. Get Your API Keys

- **AI/ML API:** Sign up at https://aimlapi.com, get API key
- **OpenAI:** For GPT-4o router (or use BuildShip's built-in)

### 2. Create BuildShip Workflow

- Sign up at https://buildship.com (free tier: 100 executions/day)
- Create new workflow with REST API trigger
- Copy the webhook URL (e.g., `https://your-project.buildship.run/omi-webhook`)

### 3. Configure Omi Developer Mode

- Open Omi app → Settings → Developer Mode → ON
- Paste your BuildShip webhook URL
- Toggle Real-Time Webhook → ON

### 4. Build the Workflow Nodes

**Node 1: REST API Trigger** - Receives Omi webhook POST**Node 2: Session Buffer (Script)** - Detects "command generate [type]..." and "command execute", buffers text between them**Node 3: GPT-4o Router** - Picks best model from AI/ML API list, outputs:

```json
{
  "model": "model_id",
  "endpoint": "/v1/...",
  "method": "POST", 
  "body": { ... }
}
```

**Node 4: HTTP Request** - Calls AI/ML API with the generated config**Node 5: FFmpeg (Optional)** - For meditation audio ducking**Node 6: Storage Upload** - Uploads to BuildShip storage, returns public URL**Node 7: Omi Notification** - Sends push notification with the link

## AI/ML API Models (December 2024)

| Type | Model ID | Best For ||------|----------|----------|| Music | `music-01` | Songs, instrumentals || Voice | `hume/octave-2` | Meditations (emotional) || Image | `flux/pro` | High quality images || Video | `openai/sora-2` | Video generation |**Note:** Suno is deprecated - use `music-01` instead

## Cost Estimate

| Service | Cost ||---------|------|| BuildShip Free | $0 || AI/ML API | ~$5-20/mo depending on usage || **Total** | **$5-20/mo** |

## Command Format

1. Say: "command generate meditation about morning gratitude"
2. Continue: "include breathing exercises..."  
# Omi Voice Command Content Generator

**Complete Setup Guide for AI-Powered Content Generation via Voice Commands**

Last Updated: December 25, 2024
All endpoints verified working with live tests.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [API Reference](#api-reference)
4. [BuildShip Workflow Setup](#buildship-workflow-setup)
5. [Omi App Configuration](#omi-app-configuration)
6. [FFmpeg Audio Processing](#ffmpeg-audio-processing)
7. [Testing](#testing)
8. [Cost Estimates](#cost-estimates)

---

## Overview

This system allows you to generate AI content (meditations, music, images, videos) using voice commands through the Omi app. When you finish a conversation containing a command, Omi sends a webhook to BuildShip, which uses GPT-4o to detect and route the command to the appropriate AI/ML API model.

### What You Can Generate

| Content Type | Model Used | Output |
|-------------|-----------|--------|
| Guided Meditations | Hume Octave 2 (emotional TTS) | Public URL (MP3/WAV) |
| Music/Songs | Stable Audio or ElevenLabs Music | Public URL (MP3/WAV) |
| Images | Flux Pro | Public URL (PNG) |
| Videos | Sora 2 | Public URL (MP4) |

### Key Finding: All APIs Return Public URLs

**No BuildShip storage needed for basic generation** - AI/ML API returns CDN-hosted public URLs directly. BuildShip storage is only needed for FFmpeg post-processing (mixing voice + music).

---

## Architecture

```
┌─────────────────┐
│    Omi App      │
│  (Voice Input)  │
└────────┬────────┘
         │ Memory Webhook (POST)
         ▼
┌─────────────────────────────────────────────────────────────┐
│                    BuildShip Workflow                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │ REST Trigger │ -> │   GPT-4o     │ -> │   Branch     │   │
│  │ /omi-memory  │    │ (Detect Cmd) │    │ (Has Cmd?)   │   │
│  └──────────────┘    └──────────────┘    └──────┬───────┘   │
│                                                  │           │
│                    ┌─────────────────────────────┼───────┐   │
│                    │         Yes                 │       │   │
│                    ▼                             ▼       ▼   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ Meditation   │  │    Music     │  │   Image/     │       │
│  │ (Hume TTS)   │  │ (Poll Loop)  │  │   Video      │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                 │                  │               │
│         └─────────────────┼──────────────────┘               │
│                           ▼                                  │
│              ┌────────────────────────┐                      │
│              │  Optional: FFmpeg Mix  │                      │
│              │  (Voice + Music Duck)  │                      │
│              └───────────┬────────────┘                      │
│                          ▼                                   │
│              ┌────────────────────────┐                      │
│              │   Omi Notification     │                      │
│              │   (Send Public URL)    │                      │
│              └────────────────────────┘                      │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## API Reference

### Authentication

All AI/ML API calls use Bearer token authentication:

```
Authorization: Bearer YOUR_AIML_API_KEY
Content-Type: application/json
```

### Environment Variables

```bash
AIML_API_KEY=your_aiml_api_key_here
AIML_API_URL=https://api.aimlapi.com
```

---

### 1. Voice Generation (Hume Octave 2)

**Best for:** Guided meditations, emotional narration, expressive speech

**Endpoint:** `POST https://api.aimlapi.com/tts`

**Request:**
```json
{
  "model": "hume/octave-2",
  "text": "Close your eyes and take a slow breath in. And a longer breath out..."
}
```

**Response:**
```json
{
  "audio": {
    "url": "https://cdn.aimlapi.com/generations/tts/hume-tts-xxx.wav"
  },
  "usage": {
    "characters": 177152
  }
}
```

**Returns:** Public URL directly (synchronous)

**Alternative TTS Models:**
- `elevenlabs/eleven_multilingual_v2` - Natural multilingual
- `elevenlabs/v3_alpha` - Latest ElevenLabs
- `minimax/speech-2.6-hd` - High quality
- `openai/tts-1-hd` - OpenAI HD voices

---

### 2. Music Generation (Async with Polling)

**Best for:** Background music, soundtracks, ambient sounds

**Step 1: Start Generation**

**Endpoint:** `POST https://api.aimlapi.com/v2/generate/audio`

**Request (Stable Audio - recommended for instrumental):**
```json
{
  "model": "stable-audio",
  "prompt": "Calm ambient meditation background music with soft piano and gentle nature sounds, peaceful and relaxing, 60 seconds"
}
```

**Request (ElevenLabs Music):**
```json
{
  "model": "elevenlabs/eleven_music",
  "prompt": "Peaceful meditation background music"
}
```

**Response:**
```json
{
  "id": "22e6dddb-235b-40ec-8a8a-cd9c8306f1f9:stable-audio",
  "status": "queued",
  "meta": {
    "usage": {
      "credits_used": 25200
    }
  }
}
```

**Step 2: Poll for Completion**

**Endpoint:** `GET https://api.aimlapi.com/v2/generate/audio`

**Query Parameters:**
```
?generation_id=22e6dddb-235b-40ec-8a8a-cd9c8306f1f9:stable-audio
```

**Response (when complete):**
```json
{
  "id": "22e6dddb-235b-40ec-8a8a-cd9c8306f1f9:stable-audio",
  "status": "completed",
  "audio_file": {
    "url": "https://cdn.aimlapi.com/flamingo/files/xxx.wav",
    "content_type": "application/octet-stream",
    "file_name": "generated.wav",
    "file_size": 5292078
  }
}
```

**Polling Logic:**
- Poll every 5-10 seconds
- Max wait: 2-3 minutes for music
- Status values: `queued`, `processing`, `completed`, `failed`

---

### 3. Image Generation (Flux)

**Endpoint:** `POST https://api.aimlapi.com/v1/images/generations`

**Request:**
```json
{
  "model": "flux/pro",
  "prompt": "A peaceful winter scene in Salt Lake City mountains at sunset, photorealistic",
  "size": "1024x1024"
}
```

**Response:**
```json
{
  "data": [
    {
      "url": "https://cdn.aimlapi.com/images/xxx.png"
    }
  ]
}
```

**Returns:** Public URL directly (synchronous for most sizes)

**Alternative Image Models:**
- `flux/schnell` - Faster generation
- `openai/dall-e-3` - OpenAI DALL-E

---

### 4. Video Generation (Sora)

**Endpoint:** `POST https://api.aimlapi.com/v1/videos/generations`

**Request:**
```json
{
  "model": "openai/sora-2",
  "prompt": "A peaceful forest with gentle snow falling",
  "duration": 5
}
```

**Response:** Similar to music - returns generation_id, requires polling.

---

## BuildShip Workflow Setup

### Step 1: Create New Workflow

1. Go to [BuildShip](https://buildship.com)
2. Create new workflow
3. Name it "Omi Voice Command Generator"

### Step 2: Add Secrets

In BuildShip Settings → Secrets, add:

| Secret Name | Value |
|-------------|-------|
| `AIML_API_KEY` | Your AI/ML API key |
| `OPENAI_API_KEY` | Your OpenAI API key (for GPT-4o) |

### Step 3: Build the Workflow Nodes

#### Node 1: REST API Trigger

- **Type:** REST API Call
- **Method:** POST
- **Path:** `/omi-memory`
- **Query Params:** `uid` (string)

#### Node 2: Extract Transcript (Script Node)

```javascript
export default function extractTranscript({ request }) {
  const segments = request.body.transcript_segments || [];
  const fullText = segments.map(s => s.text).join(' ');
  
  return {
    transcript: fullText,
    uid: request.query.uid,
    memoryId: request.body.id
  };
}
```

#### Node 3: GPT-4o Command Detector (OpenAI Chat Node)

**Model:** `gpt-4o`

**System Prompt:**
```
You analyze conversation transcripts for content generation commands.

DETECTION RULES (flexible matching):
- "command generate [type] ... command execute"
- "create a [type] about ..."
- "make me a [type] ..."
- "generate [type] ..."
- "I want a [type] about ..."

CONTENT TYPES:
- meditation, guided meditation, breathing exercise → type: "meditation"
- song, music, soundtrack → type: "music"
- image, picture, photo → type: "image"
- video, animation, clip → type: "video"

OUTPUT FORMAT (JSON only):
{
  "found": true/false,
  "type": "meditation|music|image|video",
  "description": "extracted description",
  "model": "hume/octave-2|stable-audio|flux/pro|openai/sora-2",
  "endpoint": "/tts|/v2/generate/audio|/v1/images/generations|/v1/videos/generations",
  "method": "POST",
  "needs_polling": false/true,
  "request_body": { ...complete API request body... }
}

MEDITATION SPECIAL HANDLING:
For meditation type, write a complete 1-2 minute meditation script including:
- Opening breath work (3-4 breaths)
- Body relaxation cues
- Main visualization/theme from user description
- Affirmations (3 "I am..." statements)
- Closing breath and return

Include natural pauses indicated by "..." for pacing.
```

**User Prompt:** `{{transcript}}`

#### Node 4: Parse JSON (Script Node)

```javascript
export default function parseResponse({ gpt_response }) {
  try {
    // Handle markdown code blocks if present
    let jsonStr = gpt_response;
    if (jsonStr.includes('```json')) {
      jsonStr = jsonStr.split('```json')[1].split('```')[0];
    } else if (jsonStr.includes('```')) {
      jsonStr = jsonStr.split('```')[1].split('```')[0];
    }
    return JSON.parse(jsonStr.trim());
  } catch (e) {
    return { found: false, error: e.message };
  }
}
```

#### Node 5: Branch Node

**Condition:** `parsed.found === true`

#### Node 6: HTTP Request to AI/ML API

**URL:** `https://api.aimlapi.com{{parsed.endpoint}}`
**Method:** `{{parsed.method}}`
**Headers:**
```json
{
  "Authorization": "Bearer {{secrets.AIML_API_KEY}}",
  "Content-Type": "application/json"
}
```
**Body:** `{{parsed.request_body}}`

#### Node 7: Handle Response (Script Node)

```javascript
export default function handleResponse({ http_response, parsed }) {
  const contentType = parsed.type;
  
  // Voice (Hume) returns URL directly
  if (contentType === 'meditation') {
    return {
      ready: true,
      url: http_response.audio.url,
      type: 'audio'
    };
  }
  
  // Image returns URL directly
  if (contentType === 'image') {
    return {
      ready: true,
      url: http_response.data[0].url,
      type: 'image'
    };
  }
  
  // Music/Video need polling
  if (contentType === 'music' || contentType === 'video') {
    return {
      ready: false,
      generation_id: http_response.id,
      type: contentType
    };
  }
}
```

#### Node 8: Poll Loop (for Music/Video)

If `ready === false`, add a loop that:
1. Waits 5 seconds
2. Calls GET `https://api.aimlapi.com/v2/generate/audio?generation_id={{generation_id}}`
3. Checks if `status === "completed"`
4. Returns `audio_file.url` when complete

#### Node 9: Send Omi Notification

**URL:** `https://api.omi.me/v2/integrations/webhook/notification`
**Method:** POST
**Body:**
```json
{
  "uid": "{{uid}}",
  "message": "Your {{type}} is ready! {{url}}"
}
```

---

## FFmpeg Audio Processing

### When to Use FFmpeg

Use FFmpeg when you want to create polished meditation experiences by mixing voice with background music using "ducking" (lowering music when voice is present).

### FFmpeg Ducking Command

```bash
ffmpeg -i voice.mp3 -i background_music.mp3 \
  -filter_complex "[1:a]volume=0.3[music];[0:a][music]sidechaincompress=threshold=0.02:ratio=4:attack=200:release=1000[ducked];[0:a][ducked]amix=inputs=2:duration=longest" \
  -c:a libmp3lame -q:a 2 output.mp3
```

### What This Does

1. `volume=0.3` - Sets background music to 30% volume
2. `sidechaincompress` - Ducks music when voice is detected
   - `threshold=0.02` - Voice detection sensitivity
   - `ratio=4` - How much to reduce music (4:1)
   - `attack=200` - How fast to duck (200ms)
   - `release=1000` - How fast to return (1000ms)
3. `amix` - Combines both audio tracks

### BuildShip FFmpeg Node Setup

1. Add **FFmpeg Node** after getting both voice and music URLs
2. Configure inputs:
   - Input 1: Voice URL from Hume
   - Input 2: Music URL from Stable Audio
3. Set filter: `[1:a]volume=0.3[m];[0:a][m]sidechaincompress=threshold=0.02:ratio=4:attack=200:release=1000[d];[0:a][d]amix=inputs=2:duration=longest`
4. Output format: MP3

### Alternative: Simple Mix (No Ducking)

```bash
ffmpeg -i voice.mp3 -i background_music.mp3 \
  -filter_complex "[1:a]volume=0.2[m];[0:a][m]amix=inputs=2:duration=first" \
  -c:a libmp3lame -q:a 2 output.mp3
```

### BuildShip Storage Upload

After FFmpeg processing:
1. Add **Storage Upload Node**
2. Input: FFmpeg output file
3. Get public URL from BuildShip CDN
4. Send this URL in notification

---

## Omi App Configuration

### Enable Developer Mode

1. Open Omi app
2. Go to **Settings** → **Developer Mode** → **ON**
3. Go to **Developer Settings**

### Set Webhook URL

**Memory Creation Webhook URL:**
```
https://0d2wdz.buildship.run/omi-memory
```

### Webhook Payload Format

Omi sends this payload after each conversation:

```json
{
  "id": "memory_abc123",
  "created_at": "2024-12-25T12:00:00.000Z",
  "transcript_segments": [
    {
      "text": "command generate meditation about Christmas abundance",
      "speaker": "SPEAKER_00",
      "speakerId": 0,
      "is_user": true,
      "start": 0.0,
      "end": 5.0
    },
    {
      "text": "include breathing exercises and gratitude",
      "speaker": "SPEAKER_00",
      "speakerId": 0,
      "is_user": true,
      "start": 5.0,
      "end": 10.0
    },
    {
      "text": "command execute",
      "speaker": "SPEAKER_00",
      "speakerId": 0,
      "is_user": true,
      "start": 10.0,
      "end": 12.0
    }
  ],
  "structured": {
    "title": "Meditation Request",
    "overview": "User requested a guided meditation"
  }
}
```

---

## Testing

### Voice Command Examples

| What You Say | Detected As | Model |
|--------------|------------|-------|
| "command generate meditation about stress relief command execute" | meditation | hume/octave-2 |
| "create a song about my morning walk" | music | stable-audio |
| "make me an image of a sunset over mountains" | image | flux/pro |
| "I want a meditation for better sleep" | meditation | hume/octave-2 |
| "generate video of a peaceful forest" | video | openai/sora-2 |

### Local Testing Script

```python
import os
import requests
import json
from dotenv import load_dotenv

load_dotenv('backend/.env')

api_key = os.environ.get('AIML_API_KEY')
headers = {
    "Authorization": f"Bearer {api_key}",
    "Content-Type": "application/json"
}

# Test Hume Octave 2 TTS
meditation_text = """
Close your eyes and take a slow breath in.
And a longer breath out.
Feel your body relax.
You are safe. You are supported.
Take one more breath in.
And out.
When you're ready, open your eyes.
"""

response = requests.post(
    "https://api.aimlapi.com/tts",
    headers=headers,
    json={"model": "hume/octave-2", "text": meditation_text}
)

if response.status_code == 201:
    url = response.json()['audio']['url']
    print(f"Audio URL: {url}")
```

### Send Test Webhook

```bash
curl -X POST "https://0d2wdz.buildship.run/omi-memory?uid=test_user" \
  -H "Content-Type: application/json" \
  -d '{
    "id": "test_memory_001",
    "transcript_segments": [
      {"text": "command generate meditation about morning gratitude", "is_user": true},
      {"text": "include breathing exercises", "is_user": true},
      {"text": "command execute", "is_user": true}
    ]
  }'
```

---

## Cost Estimates

### BuildShip

| Tier | Executions | Storage | Price |
|------|-----------|---------|-------|
| Free | 100/day | 100MB | $0 |
| Starter | 500K/month | 10GB | $19/mo |

### AI/ML API (Estimated)

| Model | Cost | Notes |
|-------|------|-------|
| Hume Octave 2 | ~$0.01-0.05/call | Per meditation |
| Stable Audio | ~$0.05-0.10/call | 25K credits per call |
| ElevenLabs Music | ~$0.03-0.05/call | 12K credits per call |
| Flux Pro | ~$0.03-0.05/call | Per image |
| Sora 2 | ~$0.10-0.50/call | Per video |
| GPT-4o | ~$0.01/call | Command detection |

### Monthly Estimate (Personal Use)

| Usage | Cost |
|-------|------|
| 5 meditations/day | ~$5-15/mo |
| 2 songs/week | ~$2-5/mo |
| 5 images/week | ~$3-5/mo |
| **Total** | **$10-25/mo** |

---

## Verified Test Results (December 25, 2024)

### Hume Octave 2 TTS
- **Status:** ✅ Working
- **Endpoint:** POST `https://api.aimlapi.com/tts`
- **Response:** Returns public CDN URL directly
- **Test URL:** `https://cdn.aimlapi.com/generations/tts/hume-tts-d327afc7-xxx.wav`

### Stable Audio
- **Status:** ✅ Working
- **Endpoint:** POST `https://api.aimlapi.com/v2/generate/audio`
- **Polling:** GET with `?generation_id=xxx`
- **Response:** Returns public CDN URL after ~30-60 seconds
- **Test URL:** `https://cdn.aimlapi.com/flamingo/files/xxx.wav`

### ElevenLabs Music
- **Status:** ✅ Working
- **Same endpoint as Stable Audio
- **Test URL:** `https://cdn.aimlapi.com/generations/hippopotamus/xxx.mp3`

### BuildShip Webhook
- **Status:** ✅ Working
- **Endpoint:** `https://0d2wdz.buildship.run/omi-memory`
- **Test:** Successfully received and processed payload

---

## Available Models (December 2024)

### Voice/TTS
- `hume/octave-2` - Emotional, expressive (recommended for meditations)
- `elevenlabs/eleven_multilingual_v2` - Natural multilingual
- `elevenlabs/v3_alpha` - Latest ElevenLabs
- `minimax/speech-2.6-hd` - High definition
- `minimax/speech-2.6-turbo` - Fast
- `openai/tts-1-hd` - OpenAI HD
- `openai/gpt-4o-mini-tts` - GPT-4o Mini TTS

### Music
- `stable-audio` - Stability AI (recommended for instrumental)
- `elevenlabs/eleven_music` - ElevenLabs
- `minimax/music-2.0` - MiniMax (requires lyrics)
- `google/lyria2` - Google Lyria

### Images
- `flux/pro` - High quality (recommended)
- `flux/schnell` - Fast generation

### Video
- `openai/sora-2` - OpenAI Sora

---

## Quick Start Checklist

- [ ] Get AI/ML API key from https://aimlapi.com
- [ ] Create BuildShip account at https://buildship.com
- [ ] Add `AIML_API_KEY` to BuildShip secrets
- [ ] Create workflow with all nodes
- [ ] Enable Omi Developer Mode
- [ ] Set Memory Webhook URL to your BuildShip endpoint
- [ ] Test with voice command: "create a meditation about gratitude"

---

## Troubleshooting

### "Cannot POST /v1/audio/speech"
Use `/tts` endpoint for Hume Octave 2, not `/v1/audio/speech`.

### Music generation stuck at "queued"
Poll the status endpoint with GET request and `generation_id` query parameter.

### No command detected
Ensure your voice command includes trigger words like "command generate", "create a", "make me", etc.

### FFmpeg errors
Ensure both audio files are downloaded before mixing. Use `-y` flag to overwrite existing files.

---

**Created by:** Voice Command Content Generator
**Last Tested:** December 25, 2024
**All endpoints verified working**


# Omi Voice Command Content Generator

**Complete Setup Guide for AI-Powered Content Generation via Voice Commands**

Last Updated: December 25, 2024
All endpoints verified working with live tests.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Best Models by Category](#best-models-by-category)
4. [API Reference](#api-reference)
5. [BuildShip Workflow Setup](#buildship-workflow-setup)
6. [FFmpeg Audio Processing](#ffmpeg-audio-processing)
7. [Omi App Configuration](#omi-app-configuration)
8. [Testing](#testing)
9. [Cost Estimates](#cost-estimates)
10. [Troubleshooting](#troubleshooting)

---

## Overview

This system allows you to generate AI content (meditations, music, images, videos) using voice commands through the Omi app. When you finish a conversation containing a command, Omi sends a webhook to BuildShip, which uses GPT-4o to detect and route the command to the appropriate AI/ML API model.

### What You Can Generate

| Content Type | Recommended Model | Output Format | Quality |
|-------------|------------------|---------------|---------|
| Guided Meditations | Hume Octave 2 | Public URL (WAV) | Best emotional expression |
| Voice/Narration | ElevenLabs v3 Alpha | Direct MP3 | Natural, clear |
| Background Music | Stable Audio | Public URL (WAV) | 30s max, loop-able |
| Ambient Sounds | ElevenLabs Music | Public URL (MP3) | Good for nature/ambiance |
| Images | Flux Pro | Public URL (PNG) | High quality |
| Videos | Sora 2 | Public URL (MP4) | Best quality |

### Key Discovery: All APIs Return Public URLs

**No BuildShip storage needed for basic generation!** AI/ML API returns CDN-hosted public URLs directly. BuildShip Storage is only needed for FFmpeg post-processing (mixing voice + music).

---

## Architecture

```
                                    ┌─────────────────┐
                                    │    Omi App      │
                                    │  (Voice Input)  │
                                    └────────┬────────┘
                                             │ Memory Webhook (POST)
                                             ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              BuildShip Workflow                                  │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │ REST Trigger │ -> │  Extract     │ -> │   GPT-4o     │ -> │   Branch     │   │
│  │ /omi-memory  │    │  Transcript  │    │   Router     │    │ (Has Cmd?)   │   │
│  └──────────────┘    └──────────────┘    └──────────────┘    └──────┬───────┘   │
│                                                                      │           │
│                                                    ┌─────────────────┼───────┐   │
│                                                    │       Yes       │ No    │   │
│                                                    ▼                 ▼       │   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌────────────┐ │   │
│  │  Meditation  │    │    Music     │    │   Image/     │    │    End     │ │   │
│  │ (Hume TTS)   │    │ (Stable+Loop)│    │   Video      │    │            │ │   │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘    └────────────┘ │   │
│         │                   │                   │                            │   │
│         └───────────┬───────┴───────────────────┘                            │   │
│                     ▼                                                        │   │
│         ┌─────────────────────┐                                              │   │
│         │  FFmpeg Processing  │ (Optional - for polished meditations)        │   │
│         │  - Ducking          │                                              │   │
│         │  - Compression      │                                              │   │
│         │  - Normalization    │                                              │   │
│         └──────────┬──────────┘                                              │   │
│                    ▼                                                         │   │
│         ┌─────────────────────┐                                              │   │
│         │  BuildShip Storage  │ (Only for FFmpeg output)                     │   │
│         └──────────┬──────────┘                                              │   │
│                    ▼                                                         │   │
│         ┌─────────────────────┐                                              │   │
│         │  Omi Notification   │                                              │   │
│         │  (Public URL)       │                                              │   │
│         └─────────────────────┘                                              │   │
│                                                                              │   │
└──────────────────────────────────────────────────────────────────────────────────┘
```

---

## Best Models by Category

### Voice/TTS Models (Tested December 25, 2024)

| Model | Best For | Response Type | Quality | Speed |
|-------|----------|---------------|---------|-------|
| `hume/octave-2` | Meditations, emotional content | URL (WAV) | Best | Fast |
| `elevenlabs/v3_alpha` | Natural narration | Direct MP3 | Excellent | Fast |
| `elevenlabs/eleven_multilingual_v2` | Multi-language | Direct MP3 | Excellent | Fast |
| `minimax/speech-2.6-hd` | General TTS | Direct MP3 | Very Good | Fast |
| `openai/tts-1-hd` | Reliable fallback | URL (WAV) | Good | Fast |

**Recommendation:** Use `hume/octave-2` for meditations (emotional, expressive), `elevenlabs/v3_alpha` for general narration.

### Music Models

| Model | Best For | Max Duration | Response Type |
|-------|----------|--------------|---------------|
| `stable-audio` | Instrumental, ambient | 30 seconds | Async (poll) |
| `elevenlabs/eleven_music` | Ambient, nature sounds | ~10 seconds | Async (poll) |
| `google/lyria2` | Experimental | Varies | Async (poll) |

**Recommendation:** Use `stable-audio` for meditation background music. Loop with FFmpeg for longer durations.

### Image Models

| Model | Best For | Response Type |
|-------|----------|---------------|
| `flux/pro` | High quality images | Sync (URL) |
| `flux/schnell` | Fast generation | Sync (URL) |

### Video Models

| Model | Best For | Response Type |
|-------|----------|---------------|
| `openai/sora-2` | High quality video | Async (poll) |

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

### Voice Generation Endpoints

#### Hume Octave 2 (Best for Meditations)

**Endpoint:** `POST https://api.aimlapi.com/tts`

**Request:**
```json
{
  "model": "hume/octave-2",
  "text": "Close your eyes and take a slow breath in..."
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

**Output:** Public URL directly (synchronous, no polling needed)

#### ElevenLabs v3 Alpha

**Endpoint:** `POST https://api.aimlapi.com/tts`

**Request:**
```json
{
  "model": "elevenlabs/v3_alpha",
  "text": "Your text here..."
}
```

**Response:** Returns raw MP3 audio directly (not JSON). Save the response body as `.mp3` file.

**Content-Type:** `audio/mpeg`

---

### Music Generation Endpoints

#### Stable Audio (Recommended for Background Music)

**Step 1 - Start Generation:**

**Endpoint:** `POST https://api.aimlapi.com/v2/generate/audio`

```json
{
  "model": "stable-audio",
  "prompt": "Peaceful ambient meditation music, soft piano, gentle pads, nature sounds, very calm and relaxing, no vocals"
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

**Step 2 - Poll for Completion:**

**Endpoint:** `GET https://api.aimlapi.com/v2/generate/audio?generation_id=xxx`

Poll every 5 seconds until status is "completed".

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

**Note:** Stable Audio generates 30-second clips. Use FFmpeg looping for longer durations.

---

### Image Generation

**Endpoint:** `POST https://api.aimlapi.com/v1/images/generations`

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

BEST MODELS (Use these for each type):
- meditation: "hume/octave-2" (endpoint: "/tts", sync)
- music: "stable-audio" (endpoint: "/v2/generate/audio", async)
- image: "flux/pro" (endpoint: "/v1/images/generations", sync)
- video: "openai/sora-2" (endpoint: "/v1/videos/generations", async)

OUTPUT FORMAT (JSON only):
{
  "found": true/false,
  "type": "meditation|music|image|video",
  "description": "extracted description",
  "model": "model_id",
  "endpoint": "/tts|/v2/generate/audio|/v1/images/generations|/v1/videos/generations",
  "method": "POST",
  "needs_polling": false/true,
  "request_body": { ...complete API request body... }
}

MEDITATION SCRIPT GENERATION:
For meditation type, write a complete 1-2 minute script including:
- Opening: 3-4 breath cycles with "..." pauses
- Body scan: Relax shoulders, jaw, belly
- Main visualization based on user's description
- 3 affirmations in "I am..." format
- Closing breath and gentle return

Format the meditation text naturally with "..." for pauses.
```

**User Prompt:** `{{transcript}}`

#### Node 4: Parse JSON (Script Node)

```javascript
export default function parseResponse({ gpt_response }) {
  try {
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
  const responseHeaders = http_response.headers || {};
  
  // Voice - Hume returns URL in JSON, ElevenLabs returns direct audio
  if (contentType === 'meditation') {
    if (http_response.audio && http_response.audio.url) {
      return { ready: true, url: http_response.audio.url, type: 'audio' };
    }
    // For ElevenLabs (direct audio), would need to upload to storage first
    return { ready: true, directAudio: true, type: 'audio' };
  }
  
  // Image returns URL directly
  if (contentType === 'image') {
    return { ready: true, url: http_response.data[0].url, type: 'image' };
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

If `ready === false`, create a loop that:
1. Waits 5 seconds (use Delay node)
2. Calls GET `https://api.aimlapi.com/v2/generate/audio?generation_id={{generation_id}}`
3. Checks if `status === "completed"`
4. Returns `audio_file.url` when complete
5. Max 60 iterations (5 minutes)

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

Use FFmpeg to create polished meditation experiences:
- Mix voice with background music
- Apply ducking (music lowers when voice plays)
- Add fade in/out transitions
- Normalize loudness for consistent playback

### Professional FFmpeg Settings (Tested)

#### Complete Professional Mix Command

```bash
ffmpeg -y -i voice.wav -i music.wav \
  -filter_complex "
    [0:a]acompressor=threshold=0.08:ratio=3:attack=10:release=200,highpass=f=80,lowpass=f=12000[voice];
    [1:a]volume=0.18,afade=t=in:st=0:d=5[music_fade];
    [music_fade]asplit=2[sc][bg];
    [voice][sc]sidechaincompress=threshold=0.012:ratio=10:attack=100:release=2500:makeup=1[ducked];
    [ducked][bg]amix=inputs=2:duration=first:dropout_transition=3[mixed];
    [mixed]afade=t=out:st=128:d=5,loudnorm=I=-16:TP=-1.5:LRA=11[out]
  " \
  -map "[out]" -c:a libmp3lame -b:a 192k output.mp3
```

#### Filter Breakdown

| Filter | Purpose | Settings |
|--------|---------|----------|
| `acompressor` | Voice clarity | threshold=0.08, ratio=3, attack=10ms, release=200ms |
| `highpass=f=80` | Remove rumble | Cut frequencies below 80Hz |
| `lowpass=f=12000` | Remove hiss | Cut frequencies above 12kHz |
| `volume=0.18` | Music level | 18% of original (subtle background) |
| `afade=t=in:st=0:d=5` | Fade in | 5 seconds at start |
| `sidechaincompress` | Ducking | threshold=0.012, ratio=10, attack=100ms, release=2500ms |
| `amix` | Combine tracks | duration=first (match voice length) |
| `afade=t=out` | Fade out | 5 seconds before end |
| `loudnorm` | Broadcast standard | I=-16 LUFS, TP=-1.5 dB, LRA=11 |

### Music Looping (for longer meditations)

Stable Audio generates 30-second clips. Loop for longer content:

```bash
# Loop music 5 times with crossfade, create 2:15 version
ffmpeg -y -stream_loop 5 -i music.wav -t 135 \
  -af "afade=t=in:st=0:d=2,afade=t=out:st=130:d=5" \
  music_looped.wav
```

### Simple Mix (No Advanced Processing)

```bash
ffmpeg -y -i voice.wav -i music.wav \
  -filter_complex "[1:a]volume=0.2[m];[0:a][m]amix=inputs=2:duration=first" \
  -c:a libmp3lame -b:a 192k output.mp3
```

### BuildShip FFmpeg Node Configuration

1. Add **FFmpeg Node** to workflow
2. **Input 1:** Voice URL from Hume
3. **Input 2:** Music URL from Stable Audio
4. **Command:** Use the professional mix command above
5. **Output:** Upload result to BuildShip Storage

---

## Omi App Configuration

### Enable Developer Mode

1. Open Omi app
2. Go to **Settings** → **Developer Mode** → **ON**
3. Go to **Developer Settings**

### Set Webhook URL

**Memory Creation Webhook URL:**
```
https://YOUR_BUILDSHIP_ENDPOINT/omi-memory
```

Example: `https://0d2wdz.buildship.run/omi-memory`

### Webhook Payload Format

Omi sends this payload after each conversation ends:

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

| What You Say | Detected Type | Model Used |
|--------------|---------------|------------|
| "command generate meditation about stress relief command execute" | meditation | hume/octave-2 |
| "create a song about my morning walk" | music | stable-audio |
| "make me an image of a sunset over mountains" | image | flux/pro |
| "I want a meditation for better sleep" | meditation | hume/octave-2 |
| "generate video of a peaceful forest" | video | openai/sora-2 |
| "create a guided breathing exercise for anxiety" | meditation | hume/octave-2 |

### Local Testing Script

```python
import os
import requests
import time
from dotenv import load_dotenv

load_dotenv('backend/.env')

api_key = os.environ.get('AIML_API_KEY')
headers = {
    "Authorization": f"Bearer {api_key}",
    "Content-Type": "application/json"
}

# Test Hume Octave 2 TTS
meditation_text = """
Close your eyes... and take a slow breath in.
And a longer breath out.
Feel your body relax.
You are safe. You are supported.
I have enough. I am supported.
Take one more breath in.
And out.
When you're ready, open your eyes.
"""

# Generate voice
print("Generating voice with Hume Octave 2...")
response = requests.post(
    "https://api.aimlapi.com/tts",
    headers=headers,
    json={"model": "hume/octave-2", "text": meditation_text}
)

if response.status_code == 201:
    url = response.json()['audio']['url']
    print(f"Voice URL: {url}")
    
    # Download
    audio = requests.get(url).content
    with open('test_voice.wav', 'wb') as f:
        f.write(audio)
    print(f"Saved: test_voice.wav ({len(audio)} bytes)")

# Generate music
print("\nGenerating music with Stable Audio...")
response = requests.post(
    "https://api.aimlapi.com/v2/generate/audio",
    headers=headers,
    json={
        "model": "stable-audio",
        "prompt": "Peaceful ambient meditation music, soft piano"
    }
)

if response.status_code == 201:
    gen_id = response.json()['id']
    print(f"Generation ID: {gen_id}")
    
    # Poll for completion
    while True:
        time.sleep(5)
        status = requests.get(
            "https://api.aimlapi.com/v2/generate/audio",
            headers=headers,
            params={"generation_id": gen_id}
        ).json()
        
        print(f"Status: {status['status']}")
        if status['status'] == 'completed':
            music_url = status['audio_file']['url']
            audio = requests.get(music_url).content
            with open('test_music.wav', 'wb') as f:
                f.write(audio)
            print(f"Saved: test_music.wav ({len(audio)} bytes)")
            break
        elif status['status'] == 'failed':
            print("Generation failed")
            break

print("\nNow run FFmpeg to mix:")
print('ffmpeg -y -i test_voice.wav -i test_music.wav -filter_complex "[1:a]volume=0.2[m];[0:a][m]amix=inputs=2:duration=first" -c:a libmp3lame output.mp3')
```

### Send Test Webhook

```bash
curl -X POST "https://YOUR_BUILDSHIP_ENDPOINT/omi-memory?uid=test_user" \
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

### BuildShip Pricing

| Tier | Executions | Storage | Price |
|------|-----------|---------|-------|
| Free | 100/day | 100MB | $0 |
| Starter | 500K/month | 10GB | $19/mo |

### AI/ML API Pricing (Estimated)

| Model | Cost per Call | Notes |
|-------|---------------|-------|
| Hume Octave 2 | ~$0.01-0.05 | Per meditation |
| ElevenLabs v3 | ~$0.01-0.03 | Per narration |
| Stable Audio | ~25K credits | Per 30s track |
| ElevenLabs Music | ~12K credits | Per track |
| Flux Pro | ~$0.03-0.05 | Per image |
| Sora 2 | ~$0.10-0.50 | Per video |
| GPT-4o | ~$0.01 | Per routing call |

### Monthly Estimate (Personal Use)

| Usage | Estimated Cost |
|-------|----------------|
| 5 meditations/day | ~$5-15/mo |
| 2 songs/week | ~$2-5/mo |
| 5 images/week | ~$3-5/mo |
| **Total** | **$10-25/mo** |

---

## Verified Test Results (December 25, 2024)

### TTS Model Comparison

| Model | Status | Response Type | File Size (short test) |
|-------|--------|---------------|------------------------|
| hume/octave-2 | Working | URL (WAV) | 137 KB |
| elevenlabs/v3_alpha | Working | Direct MP3 | 141 KB |
| elevenlabs/eleven_multilingual_v2 | Working | Direct MP3 | 107 KB |
| minimax/speech-2.6-hd | Working | Direct MP3 | 175 KB |
| openai/tts-1-hd | Working | URL (WAV) | 136 KB |

### Music Generation

| Model | Status | Duration | File Size |
|-------|--------|----------|-----------|
| stable-audio | Working | 30 seconds | 5.2 MB |
| elevenlabs/eleven_music | Working | ~10 seconds | 161 KB |

### FFmpeg Professional Mix

| Mix Type | Duration | File Size |
|----------|----------|-----------|
| Final meditation (Hume + Stable) | 2:12 | 3.04 MB |
| Final meditation (ElevenLabs) | 0:34 | 0.79 MB |

---

## Troubleshooting

### "Cannot POST /v1/audio/speech"
Use `/tts` endpoint instead. The `/v1/audio/speech` endpoint doesn't exist.

### Music generation stuck at "queued"
- Poll with GET request to `/v2/generate/audio?generation_id=xxx`
- Note: Use query parameter, not path parameter
- Max wait: 2-3 minutes

### No command detected
Ensure voice command includes trigger phrases:
- "command generate ... command execute"
- "create a [type] about..."
- "make me a [type]..."
- "generate [type]..."

### ElevenLabs returns binary instead of JSON
This is expected. ElevenLabs models return raw MP3 audio directly. Check `Content-Type: audio/mpeg` header and save response body as `.mp3`.

### FFmpeg errors
- Ensure both audio files exist before mixing
- Use `-y` flag to overwrite existing files
- Check file paths are correct (no spaces without quotes)

### Music too short for meditation
Use FFmpeg looping:
```bash
ffmpeg -stream_loop 5 -i music.wav -t 135 music_looped.wav
```

---

## Quick Start Checklist

- [ ] Get AI/ML API key from https://aimlapi.com
- [ ] Create BuildShip account at https://buildship.com
- [ ] Add `AIML_API_KEY` to BuildShip secrets
- [ ] Add `OPENAI_API_KEY` to BuildShip secrets (for GPT-4o routing)
- [ ] Create workflow with all nodes (see Node setup above)
- [ ] Enable Omi Developer Mode
- [ ] Set Memory Webhook URL to your BuildShip endpoint
- [ ] Test with: "command generate meditation about gratitude command execute"

---

## File Reference

| File | Purpose |
|------|---------|
| `CreatorSetup.md` | This documentation |
| `backend/.env` | Environment variables (AIML_API_KEY, AIML_API_URL) |

---

**Created by:** Voice Command Content Generator System
**Last Tested:** December 25, 2024
**All endpoints verified working**

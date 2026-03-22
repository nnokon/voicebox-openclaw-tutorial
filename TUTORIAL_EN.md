---
title: OpenClaw Local AI Voice Configuration Tutorial
date: 2026-03-22
tags:
  - openclaw
  - tts
  - voicebox
  - tutorial
  - english
---

# OpenClaw Local AI Voice Configuration Complete Guide

This tutorial will guide you through using Voicebox + a custom proxy to enable local TTS (Text-to-Speech) functionality in OpenClaw, with voice cloning support.

## Prerequisites

Before starting, ensure you have the following:

- ✅ **OpenClaw** - AI assistant application
- ✅ **AI Model** - Such as Kimi or any other model
- ✅ **Python Environment** - For running the proxy service
- ✅ **FastAPI & Uvicorn** - Python web framework (Install: `pip install fastapi uvicorn requests`)

## Phase 1: Installing and Configuring Voicebox

### Step 1: Download Voicebox

Visit [Voicebox GitHub](https://github.com/jamiepine/voicebox) to download the latest version.

### Step 2: Download TTS Model

Download Qwen's TTS model manually from [Hugging Face](https://huggingface.co).

### Step 3: Configure Voicebox Startup Parameters

After launching the Voicebox application, check the following two options in Settings:

1. **Keep server running when app closes** - Keep the background service running
2. **Allow network access** - Allow network access

> [!warning] Important
> These two settings are critical for proper operation and cannot be skipped.

---

## Phase 2: Voice Cloning (Optional but Recommended)

If you want to use a specific person's voice, you need to prepare:

- 📁 **Audio File** - A clean, clear speech sample
- 📝 **Audio Transcript** - The complete text content of that audio

### Obtaining Audio Samples

#### Method 1: Extract from Fish Audio (Recommended)

Fish Audio has a rich library of voice samples that you can extract directly:

1. Visit [Fish Audio](https://www.fish-audio.com/) and find the voice clip you want
2. Open your browser's developer tools (Press `F12`)
3. Switch to the **Network** tab
4. **Refresh the page** (Ctrl + R or Cmd + R)
5. Filter network requests by **Media** type
6. Find the corresponding `.mp3` file, right-click → **Open in new tab** or **Save as**

> [!tip] Getting the Transcript
> Copy the text content displayed on the page as the transcript for this audio. This will improve cloning quality.

#### Method 2: Use Local Recording

If it's your own voice or a friend's, simply record a clear speech sample (recommended 10-30 seconds).

---

Complete the voice cloning process in Voicebox, and the system will generate a unique **Profile ID**.

### Getting Profile ID

Voicebox runs on the `http://127.0.0.1:17493` port. Visit the following address to view cloned models:

```
http://localhost:17493/profiles
```

> [!caution] Note
> Record the model's **ID**, not the Name. You'll need the ID for subsequent configuration.

---

## Phase 3: Creating the TTS Proxy Service

### Step 4: Create Python Script

Create a file named `voicebox_proxy.py` with the following content:

```python
from fastapi import FastAPI, Request
from fastapi.responses import Response
import requests
import base64
import uvicorn

app = FastAPI()

VOICEBOX_URL = "http://localhost:17493/generate"
PROFILE_ID = "your-profile-id-here"  # ← Replace with your Profile ID

@app.post("/v1/audio/speech")
async def tts(request: Request):
    data = await request.json()
    text = data.get("input", "")

    payload = {
        "text": text,
        "profile_id": PROFILE_ID,
        "language": "en"  # Optional: supports other languages like "zh"
    }

    resp = requests.post(VOICEBOX_URL, json=payload, timeout=30)
    resp_json = resp.json()

    # Voicebox returns BASE64 encoded audio
    audio_base64 = resp_json.get("audio", "")
    audio_bytes = base64.b64decode(audio_base64)

    return Response(content=audio_bytes, media_type="audio/mpeg")


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8880)
```

### Step 5: Start the Proxy Service

```bash
python voicebox_proxy.py
```

The service will start at `http://localhost:8880`.

---

## Phase 4: Configuring OpenClaw

### Step 6: Configure TTS Settings

Execute the following commands in the terminal to configure OpenClaw:

```bash
openclaw config set messages.tts.auto "always"
openclaw config set messages.tts.provider "openai"
openclaw config set messages.tts.openai.apiKey "sk-any-value"
openclaw config set messages.tts.openai.baseUrl "http://localhost:8880/v1"
openclaw config set messages.tts.openai.model "tts-1"
openclaw config set messages.tts.openai.voice "default"
openclaw gateway restart
```

> [!tip] Parameter Explanation
> - `auto: "always"` - Generate voice for every response (perfect for storytelling)
> - `auto: "tagged"` - Only generate voice for marked content
> - `apiKey` - Can be any value, the proxy will ignore it
> - `voice` - Field is ignored, Profile ID controls the voice

### Step 7: Using TTS in OpenClaw

If you set `tts.auto` to `tagged`, use the following format to mark text for voice generation:

```
[[tts:text]]Your text to be read here[[/tts:text]]
```

---

## Phase 5: Testing and Verification

### Step 8: Test Audio Generation

Execute the following curl command in the terminal to test if the proxy is working correctly:

```bash
curl -X POST "http://localhost:8880/v1/audio/speech" \
  -H "Content-Type: application/json" \
  -d "{\"input\":\"Hello world\"}" \
  --output test.mp3
```

### Step 9: Verify Output

1. Check if `test.mp3` is in the Voicebox processing queue
2. Open `test.mp3` in your file explorer
3. Confirm the audio plays normally

> [!success] Success Sign
> If the generated MP3 file plays normally, the entire setup is configured correctly.

---

## Phase 6: Optional - Auto-startup

### Step 10: Create Startup Script

Create a file named `start_tts.bat`:

```batch
@echo off
echo Starting Voicebox + Proxy...

:: Change this to your Voicebox.exe full path
start "" /min "D:\Voicebox\Voicebox.exe"

timeout /t 8 >nul     :: Wait 8 seconds for Voicebox to fully start

:: Change this to your voicebox_proxy.py full path
start "" /min cmd /c "python D:\tts\voicebox_proxy.py"

echo Startup complete! (Window will minimize automatically)
```

> [!note] Path Modification
> Modify according to your actual installation location:
> - `D:\Voicebox\Voicebox.exe` → Your Voicebox application path
> - `D:\tts\voicebox_proxy.py` → Your proxy script path

### Step 11: Add to System Startup

1. Press `Win + R` to open the Run dialog
2. Type `shell:startup` and press Enter (opens the Startup folder)
3. Drag or copy `start_tts.bat` into this folder

✅ Done! After system restart, Voicebox and the proxy service will start automatically.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Proxy cannot connect to Voicebox | Check if Voicebox is running, confirm port 17493 is accessible |
| Poor audio quality | Re-clone the voice in Voicebox, ensure input audio is clear |
| OpenClaw not generating voice | Run the test curl command, check if proxy is responding correctly |
| Startup script fails | Check if paths are correct, ensure Python environment is available |

---

## Related Links

- 🔗 [Voicebox GitHub](https://github.com/jamiepine/voicebox)
- 🔗 [Hugging Face TTS Model](https://huggingface.co/Qwen/Qwen3-TTS-12Hz-1.7B-Base/tree/main)
- 🔗 [FastAPI Documentation](https://fastapi.tiangolo.com/)
- 🔗 [Fish Audio](https://www.fish-audio.com/)

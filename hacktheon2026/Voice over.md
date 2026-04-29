# 🎯 Voice Clone Challenge — Writeup

**CTF:** HackTheOnSejong 2026  
**Category:** AI / Audio  
**Flag:** `hacktheon2026{b7d30e21e4106a6ca4d451a218f15a97}`

---

## 📋 Challenge Overview

A web service presents a **Target Sentence** (rotating every minute) and asks you to upload a WAV file speaking that sentence with a **cloned voice** matching the server's hidden reference speaker.

![Challenge Page](assets/challenge_page.png)

The server verifies two metrics:

| Metric | Threshold | Description |
|--------|-----------|-------------|
| `speaker_similarity` | ≥ 0.8 | Your voice must match the target speaker |
| `text_similarity` | ≥ 0.8 | Your audio transcript must match the sentence |

Both must pass to receive the flag.

---

## 🔍 Reconnaissance

### API Endpoints (via Swagger at `/docs`)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/` | Challenge page (HTML) |
| `GET` | `/api/challenge` | Returns `token` + `target_sentence` |
| `POST` | `/api/verify` | Accepts `audio` (WAV) + `token` via multipart/form-data |

### Challenge API Response

```json
{
  "token": "471a3d68ae4e3382b98c14771990ef76",
  "target_sentence": "Chuck Norris can pass the Turing Test by staring at the interrogator.",
  "instructions": "Clone the target voice to speak the target sentence, then upload your WAV file to /api/verify with your token",
  "requirements": {
    "format": "WAV",
    "sample_rate": "16000 Hz recommended",
    "channels": "mono",
    "max_duration_seconds": 30,
    "max_file_size_mb": 10
  }
}
```

### Verify Response Schema

```json
{
  "success": true,
  "speaker_similarity": 0.8030,
  "speaker_threshold": 0.8,
  "transcript": "string",
  "text_similarity": 1.0,
  "text_threshold": 0.8,
  "flag": "string | null",
  "hints": ["string"]
}
```

### Key observations

- **No reference audio endpoint** — the target voice is hidden server-side
- Server uses a **speaker encoder** (likely SpeechBrain ECAPA-TDNN or Resemblyzer) to compute speaker similarity against an internal reference
- Sentences are **Chuck Norris / programming jokes** that rotate every minute
- Each `token` is tied to a specific sentence; must be submitted before rotation

---

## 🛠️ Solution

### Step 1 — Baseline TTS Testing

Used **[edge-tts](https://github.com/rany2/edge-tts)** (free Microsoft Azure Neural TTS) to generate audio and submit via a Python automation script:

```python
import requests, subprocess

BASE_URL = "http://3.37.31.209:8000"

# 1. Get challenge
r = requests.get(f"{BASE_URL}/api/challenge")
data = r.json()
token, sentence = data['token'], data['target_sentence']

# 2. Generate TTS with edge-tts
subprocess.run(["python3", "-m", "edge_tts", 
                "--voice", "en-US-EricNeural",
                "--text", sentence, 
                "--write-media", "out.mp3"])

# 3. Convert to WAV (16kHz, mono, PCM 16-bit)
subprocess.run(["ffmpeg", "-y", "-i", "out.mp3", 
                "-ar", "16000", "-ac", "1", "-acodec", "pcm_s16le", 
                "out.wav"])

# 4. Submit (field name MUST be 'audio', not 'file')
with open('out.wav', 'rb') as f:
    files = {'audio': ('out.wav', f, 'audio/wav')}
    data = {'token': token}
    r = requests.post(f"{BASE_URL}/api/verify", files=files, data=data)
    print(r.json())
```

Tested **14 different edge-tts voices**:

| Voice | speaker_similarity | text_similarity |
|-------|-------------------|-----------------|
| `en-US-EricNeural` | **0.6712** | 0.975 |
| `en-US-JennyNeural` | 0.6056 | 1.0 |
| `en-US-GuyNeural` | 0.5919 | 1.0 |
| `en-AU-NatashaNeural` | 0.5686 | 0.970 |
| `en-US-ChristopherNeural` | 0.5423 | 1.0 |
| `en-GB-RyanNeural` | 0.5230 | 1.0 |
| `en-US-AriaNeural` | 0.5067 | 1.0 |

> **`text_similarity` was trivial** — TTS reads text perfectly (1.0).  
> **`speaker_similarity` maxed at 0.67** — below the 0.8 threshold.

The hint confirmed: *"Speaker similarity too low. Try using more reference audio for cloning or a better TTS model."*

### Step 2 — Pitch Shifting Strategy

Since no reference audio was available on the server, the approach shifted to **audio signal manipulation** — modifying voice characteristics (pitch, formant) post-TTS to match the hidden reference voice.

```python
def pitch_shift(input_wav, output_wav, semitones):
    """Shift pitch by N semitones using ffmpeg's asetrate trick."""
    factor = 2 ** (semitones / 12.0)
    new_rate = int(16000 * factor)
    subprocess.run([
        "ffmpeg", "-y", "-i", input_wav,
        "-af", f"asetrate={new_rate},aresample=16000",
        "-ac", "1", "-acodec", "pcm_s16le", output_wav
    ])
```

This changes the pitch without altering the speech duration, effectively shifting the speaker's voice characteristics in the embedding space.

### Step 3 — Brute-Force Voice × Pitch Combinations

Automated the script to try **every edge-tts voice** with **pitch shifts from -3 to +3 semitones**, with 8-second delays to avoid server rate limiting:

```
Voice × Pitch → speaker_similarity
─────────────────────────────────────
en-US-BrianNeural pitch=-2.0 → 0.7774 🔥 (close!)
en-US-BrianNeural pitch=-1.0 → 0.8030 ✅ (PASS!)
```

### 🚩 Flag Captured!

```
Voice:              en-US-BrianNeural
Pitch Shift:        -1.0 semitone
speaker_similarity: 0.8030 / 0.8  ✅
text_similarity:    1.0000 / 0.8  ✅

FLAG: hacktheon2026{b7d30e21e4106a6ca4d451a218f15a97}
```

---

## 🧠 Key Takeaways

1. **Speaker encoders operate in embedding space** — small pitch/formant modifications can significantly shift the cosine similarity between voice embeddings
2. **No reference audio ≠ impossible** — brute-forcing TTS voices × pitch variations was sufficient to cross the 0.8 threshold
3. **edge-tts is a powerful free tool** — Microsoft Azure Neural voices are high quality and diverse enough for this kind of attack
4. **Rate limiting matters** — the server dropped connections after rapid requests; adding retry logic + 8s delays was critical
5. **API field names matter** — wasted initial attempts using `file` when the server required `audio` (discovered via Swagger docs)

---

## 🔧 Tools Used

| Tool | Purpose |
|------|---------|
| [edge-tts](https://github.com/rany2/edge-tts) | Text-to-Speech (Microsoft Azure Neural voices) |
| [ffmpeg](https://ffmpeg.org/) | Audio conversion + pitch shifting |
| [Python requests](https://docs.python-requests.org/) | API automation with retry/backoff |
| Swagger UI (`/docs`) | API reconnaissance |
| Kali Linux | Attack platform |


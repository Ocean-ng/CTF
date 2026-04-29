# Plottergeist — HackTheon Sejong 2026

> **Category:** Hardware / Forensics  
> **Tags:** Acoustic Side-Channel Attack, Hardware Protocol Reverse Engineering, CoreXY Kinematics  
> **Flag:** `hacktheon2026{the_plotter_reveals_its_secret_through_sound}`

---

## Challenge Description

> I built a one-of-a-kind custom CoreXY pen plotter using logic I designed myself. However, the pen plotter stopped working during the final bench test. The only remaining evidence is controller traffic logs and microphone recordings captured from the same workbench session.
>
> The capture only shows motion traffic. It does not directly tell you which moves actually put ink on the page. The microphone recording may contain small but repeatable differences between move types.
>
> Reconstruct what the machine drew and recover the flag.

**Files:** `plottergeist.pcap`, `bench_mic.wav`

---

## Solution

### Step 1 — Protocol Reverse Engineering

Opening the PCAP in Wireshark reveals UDP traffic between `10.13.37.10` (Controller) and `10.13.37.42` (Plotter) on ports `31337 ↔ 9020`.

Packet #5 from the plotter contains a plaintext format announcement:

```
c1 46 4d 54 3a 4f 50 7c 53 51 7c 44 54 7c 41 4d 7c 42 4d 7e
     F  M  T  :  O  P  |  S  Q  |  D  T  |  A  M  |  B  M  ~
```

This tells us the binary payload format: **`OP | SQ | DT | AM | BM`**, terminated by `0x7E` (`~`).

#### Packet Types

| Direction | OP byte | Size | Description |
|-----------|---------|------|-------------|
| Controller → Plotter | `0xC0` | 12B | Init: `[0xC0]["AB"][SQ:u16BE][DT:u16BE][AM:i16BE][BM:i16BE][0x7E]` |
| Controller → Plotter | `0xA1` / `0xA2` | 10B | Move: `[OP:u8][SQ:u16BE][DT:u16BE][AM:i16BE][BM:i16BE][0x7E]` |
| Plotter → Controller | `0xC1` | 20B | Format announcement |
| Plotter → Controller | `0x55` (`U`) | 8B | ACK: `[0x55][SQ:u16BE][4B data][0x7E]` |

All multi-byte integers are **big-endian**. AM and BM are **signed int16** (motor step deltas).

#### Field Meanings

- **OP** — Operation code (`0xA1` = standard move)
- **SQ** — Sequence number (incrementing)
- **DT** — Delta time / duration (in tick units)
- **AM** — Motor A step count (signed)
- **BM** — Motor B step count (signed)

### Step 2 — CoreXY Kinematics

The plotter uses CoreXY belt geometry. The conversion from motor steps to Cartesian coordinates:

```
ΔX = (ΔA + ΔB) / 2
ΔY = (ΔA - ΔB) / 2
```

Parsing all command packets and accumulating positions gives us the 2D trajectory of the pen head.

**Key observation:** In almost all commands, `AM == BM`, resulting in pure X-axis movement (`ΔY = 0`). Occasional large negative values (AM ≈ BM ≈ −350) represent carriage returns. This structure is consistent with a **raster/dot-matrix drawing style** — the plotter makes multiple horizontal passes.

### Step 3 — Acoustic Side-Channel Attack

The PCAP only contains **motion commands** — it does NOT encode pen up/down state. The challenge explicitly states:

> *"The microphone recording may contain small but repeatable differences between move types."*

When the pen is **down** (touching paper), friction produces slightly higher acoustic energy than when the pen is **up** (moving in air). We can detect this by:

1. **Synchronizing** audio time to command timeline using DT values
2. **Segmenting** the audio into per-command windows
3. **Computing RMS** (Root Mean Square) energy for each segment
4. **Visualizing** the trajectory colored by RMS intensity

### Step 4 — Visualization

The winning method turned out to be the simplest: a **scatter plot** of all pen positions, colored by the RMS energy of the corresponding audio segment.

```python
fig, ax = plt.subplots(figsize=(20, 8))
px = [positions[i+1][0] for i in range(len(commands))]
py = [positions[i+1][1] for i in range(len(commands))]
sc = ax.scatter(px, py, c=rms_values, cmap='hot', s=2)
ax.invert_yaxis()
ax.set_aspect('equal')
plt.show()
```

The result immediately reveals the flag as readable text:

- **Bright dots** (high RMS) = pen **down** → ink on paper → friction noise
- **Dark dots** (low RMS) = pen **up** → no contact → only motor noise

![Plottergeist Flag](method5_scatter.png)

### Flag

```
hacktheon2026{the_plotter_reveals_its_secret_through_sound}
```

---

## Key Takeaways

1. **Protocol RE:** The `FMT:` announcement string was a deliberate hint from the challenge author, making binary parsing straightforward.
2. **CoreXY:** Understanding that `AM ≈ BM` means pure X-axis movement was crucial to interpreting the data correctly.
3. **Side-channel:** No complex signal processing was needed — a simple RMS-colored scatter plot was enough because the acoustic difference between pen-up and pen-down is consistent and repeatable.
4. **Lesson:** In real-world scenarios, even microphone recordings from a workbench can leak sensitive information about what a CNC/plotter machine is drawing or fabricating.

---

## Solver Script

```
#!/usr/bin/env python3
"""
Voice Clone Challenge - Final Solver
=====================================
Robust script with:
  - Retry + exponential backoff (avoid rate limiting)
  - Coqui TTS VITS/VCTK multi-speaker (109 speakers)
  - Pitch-shifted edge-tts variants
  - Single-attempt mode for quick testing
"""

import requests
import subprocess
import sys
import os
import time
import json
import shutil
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

BASE_URL = "http://3.37.31.209:8000"
DELAY_BETWEEN_REQUESTS = 8  # seconds between API calls to avoid rate limit

# ============================================================
# HTTP Session with retry
# ============================================================

def get_session():
    """Create requests session with retry logic."""
    session = requests.Session()
    retry = Retry(total=3, backoff_factor=5, status_forcelist=[429, 500, 502, 503])
    adapter = HTTPAdapter(max_retries=retry)
    session.mount("http://", adapter)
    session.mount("https://", adapter)
    return session

SESSION = get_session()

def api_get(path, **kwargs):
    """GET with retry and backoff."""
    for attempt in range(3):
        try:
            r = SESSION.get(f"{BASE_URL}{path}", timeout=30, **kwargs)
            return r
        except Exception as e:
            wait = 10 * (attempt + 1)
            print(f"    ⚠️ Connection error, retrying in {wait}s... ({e.__class__.__name__})")
            time.sleep(wait)
    raise ConnectionError(f"Failed after 3 attempts")

def api_post(path, **kwargs):
    """POST with retry and backoff."""
    for attempt in range(3):
        try:
            r = SESSION.post(f"{BASE_URL}{path}", timeout=60, **kwargs)
            return r
        except Exception as e:
            wait = 10 * (attempt + 1)
            print(f"    ⚠️ Connection error, retrying in {wait}s... ({e.__class__.__name__})")
            time.sleep(wait)
    raise ConnectionError(f"Failed after 3 attempts")

# ============================================================
# Core functions
# ============================================================

def get_challenge():
    r = api_get("/api/challenge")
    data = r.json()
    print(f"  Token:    {data['token'][:16]}...")
    print(f"  Sentence: {data['target_sentence']}")
    return data['token'], data['target_sentence']

def submit(token, wav_path):
    with open(wav_path, 'rb') as f:
        files = {'audio': ('output.wav', f, 'audio/wav')}
        data = {'token': token}
        r = api_post("/api/verify", files=files, data=data)
    
    try:
        resp = r.json()
    except:
        print(f"    Raw response: {r.text[:200]}")
        return {"speaker_similarity": 0, "text_similarity": 0}
    
    spk = resp.get('speaker_similarity', 0)
    txt = resp.get('text_similarity', 0)
    success = resp.get('success', False)
    
    icon = "✅" if success else "❌"
    print(f"    {icon} speaker={spk:.4f}/0.8  text={txt:.4f}/0.8")
    if resp.get('transcript'):
        print(f"       transcript: \"{resp['transcript'][:80]}\"")
    if resp.get('flag'):
        print(f"\n    🚩🚩🚩 FLAG: {resp['flag']} 🚩🚩🚩\n")
    if resp.get('hints'):
        for h in resp['hints']:
            print(f"    💡 {h}")
    
    return resp

def to_wav_16k(src, dst):
    subprocess.run(["ffmpeg", "-y", "-i", src, "-ar", "16000", "-ac", "1", 
                     "-acodec", "pcm_s16le", dst], capture_output=True)

def tts_edge(text, voice, output):
    mp3 = output + ".mp3"
    r = subprocess.run([sys.executable, "-m", "edge_tts", "--voice", voice, 
                        "--text", text, "--write-media", mp3], capture_output=True)
    if r.returncode != 0:
        return False
    to_wav_16k(mp3, output)
    if os.path.exists(mp3): os.remove(mp3)
    return os.path.exists(output) and os.path.getsize(output) > 1000

def pitch_shift(input_wav, output_wav, semitones):
    factor = 2 ** (semitones / 12.0)
    new_rate = int(16000 * factor)
    subprocess.run(["ffmpeg", "-y", "-i", input_wav, "-af", 
                     f"asetrate={new_rate},aresample=16000",
                     "-ac", "1", "-acodec", "pcm_s16le", output_wav], capture_output=True)
    return os.path.exists(output_wav)

# ============================================================
# Coqui TTS with VCTK speakers  
# ============================================================

def setup_coqui():
    """Try to import Coqui TTS."""
    try:
        from TTS.api import TTS
        return True
    except ImportError:
        print("  Coqui TTS not installed. Installing...")
        r = subprocess.run([sys.executable, "-m", "pip", "install", "TTS", "-q"], capture_output=True)
        try:
            from TTS.api import TTS
            return True
        except:
            return False

def tts_coqui_vctk(text, speaker_id, output):
    """Generate TTS with Coqui VITS/VCTK model (109 speakers)."""
    try:
        from TTS.api import TTS
        tts = TTS("tts_models/en/vctk/vits", progress_bar=False)
        raw = output + ".raw.wav"
        tts.tts_to_file(text=text, speaker=speaker_id, file_path=raw)
        to_wav_16k(raw, output)
        if os.path.exists(raw): os.remove(raw)
        return os.path.exists(output)
    except Exception as e:
        print(f"    Coqui error: {e}")
        return False

def get_vctk_speakers():
    """Get list of VCTK speaker IDs."""
    try:
        from TTS.api import TTS
        tts = TTS("tts_models/en/vctk/vits", progress_bar=False)
        if hasattr(tts, 'speakers') and tts.speakers:
            return tts.speakers
    except:
        pass
    # Fallback: common VCTK speaker IDs
    return [f"p{i}" for i in range(225, 377)]

# ============================================================
# Main solver
# ============================================================

def try_single(voice_name, generate_fn, delay=DELAY_BETWEEN_REQUESTS):
    """Try a single voice: get challenge → generate → submit."""
    try:
        token, sentence = get_challenge()
        wav = "attempt.wav"
        
        if not generate_fn(sentence, wav):
            print(f"    ⚠️ TTS generation failed")
            return None, 0
        
        resp = submit(token, wav)
        
        if resp.get('flag') or resp.get('success'):
            return resp, 999  # Flag found!
        
        time.sleep(delay)
        return resp, resp.get('speaker_similarity', 0)
        
    except Exception as e:
        print(f"    ❌ Error: {e}")
        time.sleep(delay)
        return None, 0

def solve():
    print("=" * 65)
    print("  🎯 VOICE CLONE - FINAL SOLVER")
    print("  Rate-limit aware | Multi-engine | Multi-speaker")
    print("=" * 65)
    
    # Install base deps
    subprocess.run([sys.executable, "-m", "pip", "install", "requests", "edge-tts", "gtts", "-q"],
                   capture_output=True)
    
    results = []  # (similarity, name)
    
    # ---- PHASE 1: Edge TTS with new voices ----
    print(f"\n{'='*65}")
    print(f"  PHASE 1: Edge TTS voices ({DELAY_BETWEEN_REQUESTS}s delay between requests)")
    print(f"{'='*65}")
    
    edge_voices = [
        "en-US-EricNeural",        # best so far 0.67
        "en-US-AndrewNeural",
        "en-US-BrianNeural", 
        "en-US-RogerNeural",
        "en-US-SteffanNeural",
    ]
    
    best_edge_voice = "en-US-EricNeural"
    best_edge_sim = 0
    
    for voice in edge_voices:
        print(f"\n  📢 {voice}")
        resp, sim = try_single(voice, lambda text, out: tts_edge(text, voice, out))
        if sim == 999: return  # Flag!
        results.append((sim, voice))
        if sim > best_edge_sim:
            best_edge_sim = sim
            best_edge_voice = voice
            print(f"    ⭐ New best edge voice: {voice} ({sim:.4f})")
    
    # ---- PHASE 2: Pitch-shift best edge voice ----
    print(f"\n{'='*65}")
    print(f"  PHASE 2: Pitch-shifting {best_edge_voice} (best={best_edge_sim:.4f})")
    print(f"{'='*65}")
    
    for semitones in [-2, -1, -0.5, 0.5, 1, 2]:
        name = f"{best_edge_voice} pitch={semitones:+.1f}"
        print(f"\n  🔧 {name}")
        
        def gen_pitch(text, out, voice=best_edge_voice, st=semitones):
            base = out + ".base.wav"
            if not tts_edge(text, voice, base):
                return False
            pitch_shift(base, out, st)
            if os.path.exists(base): os.remove(base)
            return os.path.exists(out)
        
        resp, sim = try_single(name, gen_pitch)
        if sim == 999: return
        results.append((sim, name))
    
    # ---- PHASE 3: Coqui TTS VCTK speakers ----
    print(f"\n{'='*65}")
    print(f"  PHASE 3: Coqui TTS VCTK (109 speakers)")
    print(f"{'='*65}")
    
    if setup_coqui():
        speakers = get_vctk_speakers()
        print(f"  Found {len(speakers)} speakers")
        
        for i, spk in enumerate(speakers):
            print(f"\n  🎤 [{i+1}/{len(speakers)}] Speaker: {spk}")
            
            def gen_coqui(text, out, speaker=spk):
                return tts_coqui_vctk(text, speaker, out)
            
            resp, sim = try_single(f"VCTK-{spk}", gen_coqui)
            if sim == 999: return
            results.append((sim, f"VCTK-{spk}"))
            
            # If we found a good match, try pitch variations
            if sim >= 0.75:
                print(f"    🔥 Close match! Trying pitch variations...")
                for st in [-1, -0.5, 0.5, 1]:
                    def gen_coqui_pitch(text, out, speaker=spk, semitones=st):
                        base = out + ".base.wav"
                        if not tts_coqui_vctk(text, speaker, base):
                            return False
                        pitch_shift(base, out, semitones)
                        if os.path.exists(base): os.remove(base)
                        return os.path.exists(out)
                    
                    name = f"VCTK-{spk}+pitch{st:+.1f}"
                    print(f"    🔧 {name}")
                    resp, sim2 = try_single(name, gen_coqui_pitch)
                    if sim2 == 999: return
                    results.append((sim2, name))
    else:
        print("  ❌ Coqui TTS not available")
        print("  Install: pip install TTS torch torchaudio")
    
    # ---- PHASE 4: gTTS ----
    print(f"\n{'='*65}")
    print(f"  PHASE 4: Google TTS")
    print(f"{'='*65}")
    
    def gen_gtts(text, out):
        try:
            from gtts import gTTS
            mp3 = out + ".mp3"
            gTTS(text=text, lang='en', slow=False).save(mp3)
            to_wav_16k(mp3, out)
            if os.path.exists(mp3): os.remove(mp3)
            return os.path.exists(out)
        except:
            return False
    
    print(f"\n  📢 gTTS")
    resp, sim = try_single("gTTS", gen_gtts)
    if sim == 999: return
    results.append((sim, "gTTS"))
    
    # ---- SUMMARY ----
    print(f"\n{'='*65}")
    print(f"  📊 RESULTS SUMMARY (Top 20)")
    print(f"{'='*65}")
    
    results.sort(key=lambda x: x[0], reverse=True)
    for i, (sim, name) in enumerate(results[:20], 1):
        bar = "█" * int(sim * 40)
        mark = "🚩" if sim >= 0.8 else "⭐" if sim >= 0.7 else "  "
        print(f"  {mark} {i:>3}. {sim:.4f} {bar} {name}")
    
    print(f"\n  Threshold: 0.8 {'='*32}|")
    
    if results and results[0][0] >= 0.8:
        print(f"\n  ✅ Found matching voice: {results[0][1]}")
    else:
        print(f"\n  ❌ Best: {results[0][0]:.4f} ({results[0][1]})")
        print(f"  Need ≥0.8. Try providing a reference audio file.")

if __name__ == "__main__":
    solve()
```

---

*HackTheon Sejong 2026 — Forensics/Hardware*

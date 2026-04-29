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

Full solver: [`solve.py`](solve.py)

```bash
pip install scapy numpy scipy matplotlib
python solve.py
```

---

*HackTheon Sejong 2026 — Forensics/Hardware*

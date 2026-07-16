---
name: strudel-fast-skill
description: >
  Access and control a Live Strudel music coding instance via its REST API.
  Provides Strudel pattern language knowledge, synthesis / audio effect / sample techniques,
  style guides for multiple electronic music genres, and complete API calling examples.
  Use when: generating Strudel live-coding patterns, pushing code to a Strudel instance,
  managing patches, applying templates, or producing any music via the Strudel API.
  Trigger keywords: "strudel", "live code", "pattern", "push code", "apply template",
  "save patch", "strudel beat", "techno strudel", "ambient strudel", "strudel API",
  "drum pattern", "synth pattern", "strudel bass", "strudel chord".
---

# Strudel Fast Skill — Live Coding Music API

Quickly generate, manipulate, and push Strudel live-coding patterns to a Phoenix-powered
Live Strudel instance via its REST API. This skill includes the full pattern language
reference, genre-specific production techniques, and every API endpoint with real curl
examples.

## Connection Setup

All API calls target a Live Strudel base URL (e.g. `http://localhost:4000` in development,
or `https://your-live-strudel.app` in production). Every call must carry the `X-API-Key`
header.

```bash
export STRUDEL_BASE="http://localhost:4000"
export STRUDEL_API_KEY="your-secret-api-key"
```

### Response Envelope

All endpoints return JSON. Success responses contain `status: "ok"` plus a `timestamp`.
Error responses contain `status: "error"` with `error` and `reason` fields, plus an
appropriate HTTP code (400, 401, 404, 409, 429, 5xx).

---

## API Endpoints

### 1. Get Current Code

Retrieve the active Strudel code, history position, and live subscriber count.

```bash
curl -s "$STRUDEL_BASE/api/strudel/code" \
  -H "X-API-Key: $STRUDEL_API_KEY"
```

**Response:**
```json
{
  "code": "stack(\n  sound(\"bd*4\").gain(1.2),\n  sound(\"~ sd ~ sd\").gain(0.9),\n  sound(\"hh*8\").gain(0.6)\n).cpm(130)\n",
  "timestamp": "2025-08-14T09:30:00Z",
  "history": { "position": 1, "total": 7 },
  "subscribers": 3
}
```

### 2. Push Code

Push a new pattern to all connected browser clients. Optional `auto_play: true` starts
playback immediately.

```bash
curl -s -X POST "$STRUDEL_BASE/api/strudel/code" \
  -H "X-API-Key: $STRUDEL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "stack(\n  sound(\"bd*4\").gain(1.2),\n  sound(\"~ sd ~ sd\").gain(0.9),\n  sound(\"hh*8\").gain(0.6)\n).cpm(130)",
    "auto_play": true
  }'
```

**Response:**
```json
{
  "status": "ok",
  "broadcasted_to": 3,
  "code_preview": "stack(  sound(\"bd*4\"...",
  "timestamp": "2025-08-14T09:30:00Z"
}
```

**Validation Rules** (server-side `StrudelCodeValidator`):
- Maximum 10,000 characters
- Code must be a non-empty string
- Forbidden patterns: `eval(`, `Function(`, `<script`, `import(`

### 3. List Templates

List all available pattern templates with IDs, descriptions, and parameter schemas.

```bash
curl -s "$STRUDEL_BASE/api/strudel/templates" \
  -H "X-API-Key: $STRUDEL_API_KEY"
```

**Response:**
```json
{
  "templates": [
    {
      "id": "four_on_floor",
      "name": "Four on the Floor",
      "description": "Classic techno kick pattern",
      "code": "sound(\"bd*4\")",
      "params": {},
      "category_id": "drums",
      "category_name": "Drums & Percussion",
      "tags": []
    },
    {
      "id": "techno_beat",
      "name": "Techno Beat",
      "description": "Full techno drum pattern with kick, snare, hihats",
      "code": "stack(\n  sound(\"bd*4\").gain(1.2),\n  sound(\"~ sd ~ sd\").gain(0.9),\n  sound(\"hh*8\").gain(0.6)\n)",
      "params": {
        "tempo": {"type": "number", "default": 130},
        "kick_gain": {"type": "number", "default": 1.2},
        "snare_gain": {"type": "number", "default": 0.9},
        "hat_gain": {"type": "number", "default": 0.6}
      },
      "category_id": "drums",
      "category_name": "Drums & Percussion",
      "tags": []
    }
  ]
}
```

### 4. Apply Template

Generate code from a named template with optional parameter substitution.
Parameters use `{{key}}` placeholder replacement.

```bash
curl -s -X POST "$STRUDEL_BASE/api/strudel/templates/techno_beat/apply" \
  -H "X-API-Key: $STRUDEL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "params": { "tempo": 140, "kick_gain": 1.3, "snare_gain": 0.8, "hat_gain": 0.5 }
  }'
```

**Response:**
```json
{
  "status": "ok",
  "code": "stack(...)\n",
  "template_id": "techno_beat",
  "timestamp": "2025-08-14T09:30:00Z"
}
```

### 5. Undo / Redo

Navigate the code history ring buffer (up to 50 versions).

```bash
# Undo — step backward in history
curl -s -X POST "$STRUDEL_BASE/api/strudel/undo" \
  -H "X-API-Key: $STRUDEL_API_KEY"

# Redo — step forward in history
curl -s -X POST "$STRUDEL_BASE/api/strudel/redo" \
  -H "X-API-Key: $STRUDEL_API_KEY"
```

**Response (undo):**
```json
{
  "status": "undone",
  "code": "...",
  "timestamp": "2025-08-14T09:30:00Z"
}
```

If no history exists to undo, returns **HTTP 409**:
```json
{"error": "No history to undo"}
```

### 6. List Patches

List persisted code snapshots.

```bash
# User-created patches only
curl -s "$STRUDEL_BASE/api/strudel/patches" \
  -H "X-API-Key: $STRUDEL_API_KEY"

# Include built-in default patches
curl -s "$STRUDEL_BASE/api/strudel/patches?include_defaults=true" \
  -H "X-API-Key: $STRUDEL_API_KEY"
```

**Response:**
```json
{
  "patches": [
    {
      "id": 42,
      "code": "sound(\"bd*4\").cpm(120)",
      "name": "My Kick Pattern",
      "source": "api",
      "default": false,
      "inserted_at": "2025-08-14T09:00:00Z"
    }
  ]
}
```

### 7. Save Patch

Persist the current code (or provided code) as a named snapshot.

```bash
curl -s -X POST "$STRUDEL_BASE/api/strudel/patches" \
  -H "X-API-Key: $STRUDEL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "code": "sound(\"bd*4\").cpm(120)",
    "name": "My Kick Pattern"
  }'
```

**Response:**
```json
{
  "status": "ok",
  "patch_id": 42,
  "code_preview": "sound(\"bd*4\")...",
  "source": "api",
  "timestamp": "2025-08-14T09:30:00Z"
}
```

If `code` is omitted, the current active code from `StrudelState` is saved automatically.

### 8. Load Patch

Load a persisted patch by ID and push it to all connected clients.

```bash
curl -s -X POST "$STRUDEL_BASE/api/strudel/patches/42/load" \
  -H "X-API-Key: $STRUDEL_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"auto_play": true}'
```

### 9. Get Patch Detail

```bash
curl -s "$STRUDEL_BASE/api/strudel/patches/42" \
  -H "X-API-Key: $STRUDEL_API_KEY"
```

### 10. Delete Patch

```bash
# Delete a single patch (default patches cannot be deleted)
curl -s -X DELETE "$STRUDEL_BASE/api/strudel/patches/42" \
  -H "X-API-Key: $STRUDEL_API_KEY"

# Delete ALL non-default patches (bulk clear)
curl -s -X DELETE "$STRUDEL_BASE/api/strudel/patches" \
  -H "X-API-Key: $STRUDEL_API_KEY"
```

Default patches return **HTTP 409**:
```json
{"error": "Cannot delete a default patch"}
```

---

## Error Handling Quick Reference

| HTTP | Meaning | Agent Action |
|------|---------|-------------|
| 200 | Success | Proceed |
| 400 | Bad request / validation failed | Inspect `reason`, fix code, retry |
| 401 | Unauthorized / missing X-API-Key | Verify key configuration, do NOT retry blindly |
| 404 | Patch or template not found | List first, then use correct ID |
| 409 | Conflict — no history, or default patch delete | Check state before retry |
| 429 | Rate limit exceeded | Backoff 60 seconds, retry with exponential delay |
| 204 | No content (successful DELETE) | Proceed |
| 5xx | Server error | Exponential backoff, max 3 retries |

---

## Strudel Pattern Language Core

Strudel is a functional pattern language for live coding music, influenced by TidalCycles.
Every value is a **Pattern** — a time-varying stream of events. Functions compose and
transform these patterns.

### Time Structure

| Concept | Description |
|---------|-------------|
| **Cycle** | The fundamental unit of time (default = 1 bar / measure) |
| **Step** | Smallest subdivision within a cycle |
| `fast(n)` | Speed up by factor `n` (cycle plays `n` times) |
| `slow(n)` | Slow down by factor `n` |
| `.cpm(n)` | Cycles per minute — use instead of BPM for pattern speed |
| `.bpm(n)` | Alias for setting tempo |

### Mini-Notation

Compact ASCII syntax inside string arguments:

| Syntax | Meaning | Example |
|--------|---------|---------|
| `a b c` | Events in sequence, evenly spaced | `sound("bd sd hh")` |
| `a*3` | Repeat 3 times within one cycle | `sound("bd*4")` |
| `~` | Rest / silence | `sound("bd ~ bd ~")` |
| `[a b]` | Subdivide into nested pattern | `sound("bd [sd hh] bd")` |
| `<a b c>` | Rotate one event per cycle | `note("<c3 e3 g3>")` |
| `a,b,c` | Simultaneous events (chord) | `note("c3,e3,g3")` |
| `a?` | 50% chance of event | `sound("bd?")` |
| `a?0.7` | 70% chance of event | `sound("bd?0.7")` |
| `a!3` | Repeat event 3 times in one slot | `note("c3!3")` |
| `a@3` | Stretch event to 3 slots | `sound("bd@2 sd")` |

### Pattern Combinators

| Function | Description | Example |
|----------|-------------|---------|
| `stack(p1, p2, ...)` | Layer patterns simultaneously | `stack(sound("bd*4"), note("c2"))` |
| `seq(p1, p2, ...)` | Concatenate in time | `seq(sound("bd"), sound("sd"))` |
| `cat(p1, p2, ...)` | Alias for `seq` | |
| `add(p1, p2)` | Merge event streams | `add(sound("bd"), sound("sd"))` |
| `superimpose(f)` | Apply `f` alongside original | `note("c3").superimpose(add(12))` |
| `layer(f, g, ...)` | Apply multiple functions | `note("c3").layer(fast(2), rev)` |
| `struct(pat)` | Repattern events | `note("c3").struct("x ~ x x")` |

---

## Function Reference

### Sound & Synthesis

| Function | Description | Example |
|----------|-------------|---------|
| `sound(s)` | Trigger sample / synth by name | `sound("bd")` |
| `s(s)` | Alias for `sound` | `s("bd")` |
| `note(n)` | Musical note (name or MIDI) | `note("c3")`, `note(60)` |
| `freq(f)` | Frequency in Hz | `freq(440)` |
| `n(i)` | Variation index in sample bank | `sound("bd").n(3)` |
| `bank(name)` | Select sample bank | `sound("bd").bank("RolandTR808")` |
| `synth(name)` | Synth waveform | `s("sawtooth")`, `s("sine")`, `s("square")`, `s("triangle")` |
| `osc(type)` | Alias for synth selection | `osc("fm")` |

### Pattern Transformers

| Function | Description | Example |
|----------|-------------|---------|
| `fast(n)` | Speed up by `n` | `sound("bd").fast(2)` |
| `slow(n)` | Slow down by `n` | `sound("bd").slow(2)` |
| `rev` | Reverse pattern | `sound("bd sd").rev` |
| `chop(n)` | Chop into `n` slices | `sound("break").chop(8)` |
| `striate(n)` | Alias for `chop` | |
| `jux(f)` | Apply `f` to one channel only | `note("c3").jux(add(7))` |
| `pick(n)` | Randomly pick `n` events | `note("c4 d4 e4 g4").pick(4)` |
| `degrade` | Randomly drop events (50%) | `sound("bd*8").degrade` |
| `degradeBy(p)` | Drop with probability `p` | `sound("bd*8").degradeBy(0.3)` |
| `every(n, f)` | Apply `f` every `n` cycles | `sound("bd*4").every(4, rev)` |
| `every(n, f, g)` | Apply `f` every `n`, else `g` | |
| `sometimes(f)` | Randomly apply `f` (50%) | `sound("bd").sometimes(fast(2))` |
| `sometimesBy(p, f)` | Apply `f` with probability `p` | |
| `off(time, f)` | Offset a transformed copy | `note("c3").off(0.25, add(7))` |
| `palindrome` | Play forward then backward | `sound("bd sd hh").palindrome` |

### Audio Effects

| Function | Description | Range | Example |
|----------|-------------|-------|---------|
| `gain(v)` | Volume (0.0–2.0) | 0–2+ | `.gain(0.8)` |
| `lpf(freq)` | Low-pass filter cutoff | Hz | `.lpf(1200)` |
| `hpf(freq)` | High-pass filter cutoff | Hz | `.hpf(200)` |
| `bpf(freq)` | Band-pass filter | Hz | `.bpf(1000)` |
| `cutoff(freq)` | Filter cutoff (alias) | Hz | `.cutoff(800)` |
| `resonance(q)` | Filter resonance | 0–20 | `.resonance(8)` |
| `attack(t)` | Attack time (seconds) | 0–∞ | `.attack(0.01)` |
| `decay(t)` | Decay time (seconds) | 0–∞ | `.decay(0.3)` |
| `sustain(l)` | Sustain level (0–1) | 0–1 | `.sustain(0.5)` |
| `release(t)` | Release time (seconds) | 0–∞ | `.release(0.5)` |
| `delay(t)` | Delay time (seconds) | 0–1 | `.delay(0.25)` |
| `delaytime(t)` | Alias | | `.delaytime(0.375)` |
| `delayfeedback(v)` | Delay feedback (0–1) | 0–1 | `.delayfeedback(0.5)` |
| `room(v)` | Reverb room size (0–1) | 0–1 | `.room(0.6)` |
| `roomsize(v)` | Alias | | `.roomsize(0.8)` |
| `orbit(i)` | Separate effect bus | int | `.orbit(1)` |
| `dist(v)` | Distortion amount | 0–1+ | `.dist(0.5)` |
| `crush(v)` | Bit crusher (bit depth) | 1–16 | `.crush(4)` |
| `coarse(v)` | Sample rate reduction | int | `.coarse(4)` |
| `pan(v)` | Stereo pan (-1 to 1) | -1–1 | `.pan(sine)` |
| `speed(v)` | Playback speed / pitch | ratio | `.speed(1.5)` |
| `vib(v)` | Vibrato depth | 0–1 | `.vib(0.3)` |
| `vibmod(v)` | Vibrato rate | Hz | `.vibmod(5)` |
| `shape(v)` | Wave shaping | 0–1 | `.shape(0.5)` |

### Sampling

| Function | Description | Example |
|----------|-------------|---------|
| `bank(name)` | Select sample bank | `.bank("RolandTR808")` |
| `n(i)` | Sample variation index | `.n(2)` |
| `slice(i, n)` | Slice sample into `n` parts, play `i`-th | `.slice(3, 8)` |
| `begin(t)` | Start point (0–1) | `.begin(0.25)` |
| `end(t)` | End point (0–1) | `.end(0.75)` |
| `loop(t)` | Loop count | `.loop(4)` |
| `speed(r)` | Playback speed ratio | `.speed(0.5)` (half speed, down octave) |
| `unit("rate\|semitone\|cents")` | Speed unit | `.speed(12).unit("semitone")` |

### Continuous Signals (Modulators)

Use these as values in effect parameters:

| Signal | Description | Example |
|--------|-------------|---------|
| `sine` | Sine wave LFO (0–1) | `.lpf(sine.range(400, 2000).slow(4))` |
| `tri` | Triangle wave LFO | `.pan(tri)` |
| `saw` | Sawtooth LFO | `.cutoff(saw.range(200, 2000))` |
| `square` | Square wave LFO | `.gain(square.range(0.5, 1))` |
| `rand` | Random value per event | `.gain(rand)` |
| `irand(max)` | Random integer 0..max-1 | `.n(irand(5))` |
| `range(min, max)` | Scale signal to range | `sine.range(200, 800)` |
| `slow(n)` | Slow signal by factor | `sine.slow(8)` |
| `fast(n)` | Speed up signal | `rand.fast(4)` |

---

## Music Style Guides

### Techno / House

**Signature elements:**
- **Four-on-the-floor kick** — `sound("bd*4")` or `sound("bd").bank("RolandTR808")`
- **Open hi-hat on the offbeat** — `sound("~ oh ~ oh")`
- **Sidechain ducking** — Emulated by `gain(sine.range(0.4, 1).slow(4))` on non-kick elements
- **Acid bassline** — `note("c2 [c2 ~] eb2 [~ g2] f2 [f2 ~]").s("sawtooth").resonance(8).cutoff(sine.range(800, 3000).slow(4)).decay(0.3).sustain(0)`
- **Hi-hat shuffle** — `sound("hh*8").sometimes(fast(2))`

**Complete example:**
```javascript
stack(
  // Kick
  sound("bd*4").bank("RolandTR808").gain(1.3),
  // Snare on 2 and 4
  sound("~ sd ~ sd").bank("RolandTR808").gain(0.9),
  // Closed hi-hats
  sound("hh*8").bank("RolandTR808").gain(0.6).pan(0.3),
  // Open hi-hat
  sound("~ oh ~ oh").bank("RolandTR808").gain(0.5).pan(0.7),
  // Acid bassline
  note("c2 [c2 ~] eb2 [~ g2] f2 [f2 ~]").s("sawtooth")
    .resonance(8).cutoff(sine.range(800, 3000).slow(4))
    .decay(0.3).sustain(0).gain(0.8),
  // Arpeggio stab
  note("c4 e4 g4 c5").s("sawtooth").lpf(2000)
    .attack(0.01).release(0.2).gain(0.4)
).cpm(130)
```

### Ambient / Drone

**Signature elements:**
- **Long envelopes** — `.attack(2).release(4)`
- **Generative structures** — `.pick()`, `degradeBy()`, `rand`
- **Reverb-drenched pads** — `.room(0.8).roomsize(0.9)`
- **FM pads** — `s("triangle")` with slow LFO modulation
- **Drone bass** — Very low notes with long release

**Complete example:**
```javascript
stack(
  // FM drone pad
  note("c3,e3,g3").s("triangle")
    .attack(2).decay(1).sustain(0.6).release(4)
    .lpf(sine.range(600, 2000).slow(16))
    .room(0.8).roomsize(0.9)
    .gain(0.4),
  // Generative melody
  note("c4 d4 e4 g4 a4 c5").pick(4)
    .s("sine")
    .lpf(sine.range(1000, 3000).slow(8))
    .delay(0.5).delayfeedback(0.6)
    .room(0.5).gain(0.3),
  // Sub bass drone
  note("c1").s("sine")
    .attack(3).release(5)
    .lpf(200).gain(0.6),
  // Textural noise
  sound("~").s("noise")
    .lpf(sine.range(200, 2000).slow(8))
    .gain(sine.range(0.05, 0.2).slow(4))
).cpm(80)
```

### Breakbeat / Jungle

**Signature elements:**
- **Amen break slicing** — `sound("break").slice(0, 8)` or `chop(8)`
- **Time-stretch artifacts** — `.speed("1 1.5 0.75 2")`
- **Roller sub-bass** — `note("c1 [~ c1] ~ c1").s("sine").lpf(200)`
- **Fast hi-hat rolls** — `sound("hh*16")` or `sound("hh*32")`
- **Reese bass** — Detuned sawtooth with heavy modulation

**Complete example:**
```javascript
stack(
  // Amen break (sliced)
  sound("amen").slice("0 1 2 3 4 5 6 7".pick(8))
    .speed("1 1.2 0.9 1.5".pick(4))
    .gain(0.9),
  // Layered kick
  sound("bd*2").bank("RolandTR808").gain(1.1),
  // Fast hat rolls
  sound("hh*16").bank("RolandTR808")
    .gain(sine.range(0.3, 0.8).fast(4))
    .pan(rand),
  // Reese bass
  note("c1 [~ c1] ~ c1").s("sawtooth")
    .cutoff(sine.range(200, 1000).slow(2))
    .resonance(10)
    .gain(0.7),
  // FX riser every 8 bars
  note("c3").s("sawtooth")
    .cutoff(sine.range(200, 8000).slow(2))
    .resonance(10)
    .gain(sine.range(0, 0.8).slow(2))
    .every(8, id)
).cpm(160)
```

### Jazz / Fusion

**Signature elements:**
- **Walking bass** — Quarter-note bassline with chromatic approach
- **Extended chords** — 7th, 9th, 11th via mini-notation `,` syntax
- **Swing feel** — Emulated with uneven subdivisions or `.struct()`
- **Modal interchange** — Borrow chords from parallel modes
- **Brush drums** — Softer drum textures

**Complete example:**
```javascript
stack(
  // Walking bass
  note("<c2 e2 g2 bb2 d3 c3 bb2 g2>").s("sawtooth")
    .lpf(sine.range(400, 1200).slow(8))
    .gain(0.7),
  // ii-V-I chord stabs
  note("<d3,f3,a3,c4 g2,bb2,d3,f3 c3,e3,g3,bb3>").s("sawtooth")
    .attack(0.02).decay(0.3).sustain(0.3).release(0.4)
    .lpf(1500).gain(0.5),
  // Ride cymbal swing pattern
  sound("[~ ride]*4").bank("RolandTR808")
    .gain(0.5).pan(0.7),
  // Brush snare on 2 and 4
  sound("~ [~ sn:1] ~ [~ sn:1]").bank("RolandTR808")
    .gain(0.4).room(0.3),
  // Comping hits
  note("<~ e4 ~ g4 ~ a4 ~ bb4>").s("triangle")
    .attack(0.01).release(0.3)
    .lpf(3000).gain(0.3)
).cpm(120)
```

### Algorithmic Composition

**Techniques:**

#### Euclidean Rhythms
Generate evenly distributed onsets across `n` steps with `k` hits:
```javascript
// Euclidean(5,8) — 5 hits in 8 steps = x.xx.x.x
sound("bd(5,8)").bank("RolandTR808")
// Euclidean(3,7) — 3 hits in 7 steps
sound("sd(3,7)").bank("RolandTR808")
```

#### Markov Chains (simulated)
```javascript
// State transition simulated via pick
note("c4 d4 e4 g4 a4 c5".pick(8))
  .s("sine").lpf(2000).delay(0.3).gain(0.5)
```

#### Cellular Automata Texture
```javascript
// Binary pattern mapped to onsets
sound("bd(3,8) sd(2,7) hh(5,8)")
  .bank("RolandTR808")
  .gain(rand.range(0.3, 1))
```

#### Recursive / Self-Similar
```javascript
// Nested fast/slow creates fractal rhythms
sound("bd")
  .fast(2).every(4, slow(2))
  .every(8, fast(2))
  .bank("RolandTR808")
```

#### Generative Ambient
```javascript
stack(
  note("c3 e3 g3 bb3 d4".pick(6))
    .s("triangle").attack(2).release(4)
    .lpf(sine.range(400, 2000).slow(16))
    .room(0.8).gain(0.4),
  sound("~".pick(4))
    .s("noise").lpf(rand.range(200, 2000))
    .gain(rand.range(0.02, 0.1))
).cpm(60)
```

---

## Production Tips

### Gain Staging
- Keep kick around `gain(1.0–1.3)`; it is the loudest element
- Bass: `gain(0.6–0.8)` to leave headroom
- Hihats: `gain(0.4–0.6)`
- Pads / ambience: `gain(0.3–0.5)`

### Polyphony & Layering
- Use `stack()` to layer elements; each element is independent
- Use `orbit(n)` to separate effect buses (e.g. drums `orbit(0)`, synths `orbit(1)`)
- `add()` merges streams without creating new channels

### Filter Sweeps
- `lpf(sine.range(200, 8000).slow(4))` creates a smooth 4-cycle sweep
- `cutoff(saw.range(400, 4000))` creates a hard ramp
- Combine with `resonance(8–12)` for "screaming" filter peaks

### Reverb & Delay
- Short slapback: `.delay(0.125).delayfeedback(0.3).room(0.2)`
- Cavernous ambient: `.room(0.8).roomsize(0.9).delay(0.5).delayfeedback(0.7)`
- Dub-style: `.delay(0.375).delayfeedback(0.8).room(0.6)`

### Sidechain Emulation
Apply to non-kick elements:
```javascript
sound("pad").gain(sine.range(0.3, 1).slow(4))  // ducks every cycle
```

---

## Built-in Template Quick Reference

| ID | Category | Description | Key Params |
|----|----------|-------------|------------|
| `four_on_floor` | Drums | Techno kick | — |
| `techno_beat` | Drums | Full techno kit | `tempo`, `kick_gain`, `snare_gain`, `hat_gain` |
| `breakbeat` | Drums | Breakbeat style | — |
| `drum_fill` | Drums | Transition fill | — |
| `walking_bass` | Bass | Jazz walking line | `root`, `style` |
| `acid_bass` | Bass | TB-303 style | `resonance`, `cutoff_range` |
| `sub_bass` | Bass | Deep sub | `note` |
| `chord_progression` | Chords | Parameterized chords | `progression`, `root`, `synth` |
| `ambient_pad` | Chords | Ethereal pad | `chord`, `attack`, `release` |
| `arpeggio` | Melody | Rising arpeggio | `root`, `pattern` |
| `random_melody` | Melody | Generative melody | `scale`, `octave` |
| `riser` | FX | Build-up riser | `duration` |
| `noise_texture` | FX | Filtered noise | — |
| `hello_strudel` | Starters | Simple tone | — |
| `simple_beat` | Starters | Kick + bass | — |
| `full_starter` | Starters | Drums + bass + melody | — |

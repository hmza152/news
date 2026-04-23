# News-to-Reel: AI Architecture

A pipeline that takes a short raw news report (e.g. an Urdu crime story from Slack) and returns a 1–2 minute vertical (9:16) news reel with an AI reporter, b-roll visuals, captions, and lip-synced narration.

This document covers the **AI/ML perspective only** — orchestration, infra hardening, and publishing integrations are summarized but not detailed.

---

## 1. Architecture Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                       INPUT: Raw news report (Urdu)                  │
│         (text from Slack / RSS / scraper / manual paste)             │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  1. PRE-PROCESSING & NORMALIZATION                                   │
│     • Language detect, clean boilerplate ("پشاور(جنرل رپورٹر)")      │
│     • De-duplicate Story 1 / Story 2 / Story 3 → single canonical    │
│       story object {headline, lede, key_facts[], quotes[], entities} │
│     • Optional Urdu→English mirror for downstream models that need it│
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  2. SCRIPT GENERATION  (LLM — Claude / GPT-4 class)                  │
│     Output is a STRUCTURED script, not prose:                        │
│     [                                                                │
│       { scene_id, duration_s, narration_urdu,                        │
│         on_screen_text, b_roll_prompt, shot_type, mood }, ...        │
│     ]                                                                │
│     • Style guide prompt: 1–2 min, ~150–180 Urdu words, hook-first,  │
│       neutral tone, no speculation beyond source                     │
│     • Guardrails: named-entity check vs source, no hallucinated      │
│       facts, sensitivity filter (this is a graphic crime story)      │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
   ┌──────────────────┐ ┌──────────────┐ ┌────────────────────┐
   │ 3a. VISUAL PLAN  │ │ 3b. TTS      │ │ 3c. CAPTIONS /     │
   │    (per scene)   │ │  (Urdu voice)│ │     LOWER-THIRDS   │
   │                  │ │              │ │                    │
   │ • T2I prompts    │ │ ElevenLabs / │ │ Burned-in Urdu     │
   │   from b_roll_   │ │ Coqui / Bark │ │ subs + headline    │
   │   prompt         │ │ — cloned     │ │ banners            │
   │ • Negative       │ │   reporter   │ │                    │
   │   prompts (gore, │ │   voice      │ │ Force-aligned to   │
   │   minors, etc.)  │ │              │ │ TTS (whisperX)     │
   │ • Style ref      │ │ Outputs WAV  │ │                    │
   │   image for      │ │ + per-word   │ │                    │
   │   consistency    │ │   timestamps │ │                    │
   └────────┬─────────┘ └──────┬───────┘ └─────────┬──────────┘
            │                  │                   │
            ▼                  │                   │
   ┌──────────────────┐        │                   │
   │ 4. IMAGE GEN     │        │                   │
   │  Flux / SDXL /   │        │                   │
   │  Imagen / Ideo.  │        │                   │
   │                  │        │                   │
   │ + Ken Burns /    │        │                   │
   │   I2V (Runway,   │        │                   │
   │   Kling, Pika)   │        │                   │
   │   for motion     │        │                   │
   └────────┬─────────┘        │                   │
            │                  │                   │
            │        ┌─────────▼──────────┐        │
            │        │ 5. REPORTER AVATAR │        │
            │        │   + LIP SYNC       │        │
            │        │                    │        │
            │        │ Option A (SaaS):   │        │
            │        │   HeyGen / D-ID /  │        │
            │        │   Synthesia        │        │
            │        │                    │        │
            │        │ Option B (OSS):    │        │
            │        │   base clip + Wav2 │        │
            │        │   Lip / MuseTalk / │        │
            │        │   LatentSync       │        │
            │        │                    │        │
            │        │ Output: PIP-ready  │        │
            │        │ talking-head MP4   │        │
            │        └─────────┬──────────┘        │
            │                  │                   │
            └──────────────────┼───────────────────┘
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  6. VIDEO COMPOSITOR  (Remotion / MoviePy / FFmpeg)                  │
│     • 9:16 1080×1920 timeline                                        │
│     • Layer order: bg b-roll → reporter PIP → captions → bg music    │
│       → logo/branding → transitions                                  │
│     • Scene cuts driven by TTS word timestamps                       │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
┌──────────────────────────────────────────────────────────────────────┐
│  7. QA GATE  (auto + optional human-in-the-loop)                     │
│     • Vision-LLM watches the rendered MP4: factual drift, faces of   │
│       real people misused, gore, brand check                         │
│     • Loudness norm (-14 LUFS), duration in [60, 120]s               │
└──────────────────────────────┬───────────────────────────────────────┘
                               │
                               ▼
                ┌──────────────────────────────┐
                │  OUTPUT: MP4 reel (9:16)     │
                │  → S3 / Drive / IG publisher │
                └──────────────────────────────┘

Cross-cutting: orchestrator (Temporal / Prefect / simple FastAPI +
queue), per-module cache (same script → reuse images/voice), cost &
latency telemetry, prompt/version registry.
```

---

## 2. Module-by-Module Breakdown

### 1. Pre-processing & Normalization
**Input:** raw text blob (often multi-story, with bylines/boilerplate).
**Output:** canonical `Story` object.
```json
{
  "headline": "...",
  "lede": "...",
  "key_facts": ["...", "..."],
  "quotes": [{ "speaker": "...", "text": "..." }],
  "entities": { "people": [], "places": [], "orgs": [] },
  "language": "ur",
  "source_text": "..."
}
```
- LLM-assisted, with a deterministic schema validator on top.
- Strip repeated bylines like `پشاور(جنرل رپورٹر)`, merge "Story 2 / Story 3" follow-ups into a single fact set.

### 2. Script Generation (LLM)
- **Model:** Claude Opus / Sonnet 4.x or GPT-4 class.
- **Output schema (per scene):**
  ```json
  {
    "scene_id": 3,
    "duration_s": 8,
    "narration_urdu": "...",
    "on_screen_text": "...",
    "b_roll_prompt": "wide shot of a quiet Peshawar graveyard at dawn, somber, cinematic",
    "shot_type": "wide | medium | close-up | reporter",
    "mood": "somber | tense | neutral"
  }
  ```
- **Guardrails:**
  - Entity faithfulness: every named person/place must appear in source.
  - No speculation: no claims beyond the source text.
  - Sensitivity filter: no graphic descriptions of violence, no naming of minors.
  - Length budget: total `sum(duration_s)` ∈ [60, 120].

### 3a. Visual Plan
- Convert each scene's `b_roll_prompt` → final T2I prompt with style suffix (e.g. *"news documentary, grainy, muted palette, 9:16"*).
- Maintain a **style anchor**: one reference image fed back as IP-Adapter / style ref so all scenes feel like one piece.
- Negative prompts: `gore, blood, weapons, children, deformed faces, watermark, text`.

### 3b. TTS (Urdu)
- **SaaS:** ElevenLabs multilingual v2 (good Urdu, clones reporter voice from a 1–2 min sample).
- **OSS:** XTTS-v2 / Coqui fine-tuned on Urdu, or Bark.
- Emit per-word timestamps (forced-align with `whisperX` if the TTS doesn't provide them).

### 3c. Captions / Lower-thirds
- Burned-in Urdu subtitles (RTL, Noto Nastaliq Urdu font).
- Headline banner at top, source attribution at bottom.
- Driven by TTS word timestamps for kinetic typography.

### 4. Image Gen + I2V
- **T2I:** Flux.1 / SDXL / Imagen 3 / Ideogram (Ideogram for any in-image Urdu text).
- **I2V (motion):** Runway Gen-3, Kling, Pika — turn each still into a 4–6s clip with subtle camera move. Cheaper fallback: Ken Burns pan/zoom in the compositor.

### 5. Reporter Avatar + Lip Sync
- **Option A — SaaS (recommended for v1):**
  - HeyGen / D-ID / Synthesia. Provide TTS audio + chosen avatar → returns a talking-head MP4 with PIP-ready transparent/green background.
- **Option B — OSS (cheaper at scale, more work):**
  - Pre-record a 30s base clip of the "anchor" → drive lip motion with **Wav2Lip**, **MuseTalk**, or **LatentSync**.
  - Urdu phoneme handling is the main quality risk.

### 6. Video Compositor
- **Tool:** Remotion (React-based, great for templated news reels) or MoviePy / FFmpeg for headless rendering.
- **Timeline (9:16, 1080×1920, 30fps):**
  - Layer 0: background b-roll (image+motion or I2V clip)
  - Layer 1: reporter PIP (bottom-right or full-screen for intro/outro)
  - Layer 2: captions / kinetic text
  - Layer 3: lower-third banners + logo
  - Layer 4: background music bed (-20 dB under VO)
- Scene cuts triggered by `narration_urdu` word timestamps so visuals land on emphasis words.

### 7. QA Gate
- **Auto checks:**
  - Duration ∈ [60, 120] s
  - Audio loudness normalized to -14 LUFS (IG spec)
  - Vision-LLM pass: "Does this video contain any of: gore, real-person impersonation, on-screen text errors, brand misuse?"
  - Entity check: every entity in script appears in source story.
- **Optional human-in-the-loop** for sensitive categories (crime, politics, minors).

---

## 3. Development Time Estimates

Assumes **1 mid/senior ML engineer**, end-to-end production-quality module (not a throwaway demo).

| # | Module | SaaS path | OSS path |
|---|---|---|---|
| 1 | Pre-processing / story normalizer | 2–3 days | 2–3 days |
| 2 | LLM script generator (structured + guardrails + Urdu style) | 1–1.5 weeks | 1–1.5 weeks |
| 3a | Visual prompt planner (style consistency) | 3–5 days | 3–5 days |
| 3b | Urdu TTS + word timestamps | 3–5 days (ElevenLabs) | 2–3 weeks (XTTS fine-tune) |
| 3c | Captions / lower-thirds (whisperX align + RTL burn-in) | 2–3 days | 2–3 days |
| 4 | Image gen + I2V motion | 1 week | 1.5 weeks |
| 5 | Reporter avatar + lip sync | 3–5 days (HeyGen/D-ID) | 3–4 weeks (Wav2Lip/MuseTalk) |
| 6 | Compositor (Remotion / MoviePy timeline) | 1.5–2 weeks | 1.5–2 weeks |
| 7 | QA gate (vision-LLM + audio/duration checks) | 3–5 days | 3–5 days |
| – | Orchestrator, caching, retries, telemetry | 1–1.5 weeks | 1–1.5 weeks |
| – | Integration, eval harness, prompt iteration on real stories | 1–2 weeks | 1–2 weeks |

**Totals**
- **Fast path (mostly SaaS):** ~**5–7 weeks** to a usable v1.
- **OSS / self-hosted path:** ~**10–14 weeks**, dominated by lip-sync quality and Urdu TTS tuning.

---

## 4. Biggest Risks / Time-Sinks

1. **Lip-sync quality on Urdu phonemes** (OSS path) — visible artifacts will kill the format.
2. **Visual consistency across scenes** — same location/character look across 8–12 generated images is non-trivial.
3. **Factual faithfulness of the LLM script** — needs a real eval set with held-out stories, not vibes.
4. **Sensitivity handling** for crime/graphic stories — must be a separate guardrail layer, not bolted on at the end.
5. **Urdu typography (RTL, Nastaliq)** — most caption/compositor tooling is LTR-first; budget extra time.

---

## 5. Recommended v1 Stack

- **Script LLM:** Claude Sonnet 4.x (cheap + strong instruction following + good Urdu).
- **TTS:** ElevenLabs Multilingual v2 with cloned reporter voice.
- **Images:** Flux.1 [pro] for b-roll, Ideogram for any in-image Urdu text.
- **Motion:** Kling or Runway Gen-3 for hero shots; Ken Burns elsewhere.
- **Avatar + lip sync:** HeyGen (instant avatar from a 1-min recording).
- **Compositor:** Remotion (templated 9:16 news layout, easy to iterate visually).
- **Captions:** whisperX for forced alignment + Remotion `<Word>` components.
- **Orchestrator:** FastAPI + Redis queue + Temporal for retries; one `Job` per story.
- **Storage:** S3 for assets, Postgres for job/story metadata, prompt version registry.

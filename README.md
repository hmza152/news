# News-to-Reel Architecture Pack

Design notes for an AI pipeline that turns a short Urdu news report into a 1–2 minute vertical (9:16) news reel with an AI reporter.

## Files

- **[ARCHITECTURE.md](ARCHITECTURE.md)** — Full architecture write-up: ASCII diagram, module-by-module breakdown, recommended stack, risks.
- **[diagram.txt](diagram.txt)** — Standalone ASCII pipeline diagram.
- **[script_schema.json](script_schema.json)** — JSON Schema for the structured script output of the LLM module.
- **[timeline_estimates.md](timeline_estimates.md)** — Per-module dev-time estimates (SaaS vs OSS path) and critical path.

## TL;DR

```
News text → [Pre-proc] → [LLM script] → [TTS + Images + Captions] →
            [Reporter avatar + lip sync] → [Compositor] → [QA gate] → MP4 reel
```

- **Fast path (SaaS-heavy: ElevenLabs + HeyGen + Flux + Remotion):** ~5–7 weeks to v1.
- **OSS path (XTTS + Wav2Lip/MuseTalk + SDXL):** ~10–14 weeks.

Biggest time sinks: lip-sync quality on Urdu, visual consistency across scenes, factual-faithfulness evals, and RTL/Nastaliq typography.

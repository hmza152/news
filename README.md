# News-to-Reel Architecture Pack

Design notes for an AI pipeline that turns a short Urdu news report into a 1–2 minute vertical (9:16) news reel with an AI reporter. **SaaS path only** (ElevenLabs + HeyGen + Flux + Remotion).

## Files

- **[diagram.txt](diagram.txt)** — Standalone ASCII pipeline diagram.
- **[timeline_estimates.md](timeline_estimates.md)** — Per-module dev-time estimates and critical path.

## TL;DR

```
News text → [Pre-proc] → [LLM script] → [TTS + Images + Captions] →
            [Reporter avatar + lip sync] → [Compositor] → [QA gate] → MP4 reel
```

- **SaaS path (ElevenLabs + HeyGen + Flux + Remotion):** ~5–7 weeks to v1.

Biggest time sinks: visual consistency across scenes, factual-faithfulness evals, sensitivity guardrails, and RTL/Nastaliq typography.

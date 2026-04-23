# Development Timeline — News-to-Reel Pipeline

Assumes **1 mid/senior ML engineer**, working module quality (not throwaway demos).

## Module Estimates

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

## Totals

- **Fast path (mostly SaaS):** ~**5–7 weeks** to a usable v1.
- **OSS / self-hosted path:** ~**10–14 weeks**, dominated by lip-sync quality and Urdu TTS tuning.

## Critical Path (SaaS v1)

```
Week 1:    [Pre-proc] → [Script gen v0] → [TTS wired]
Week 2:    [Script gen v1 + guardrails] → [Image gen + style anchor]
Week 3:    [Avatar/lip sync (HeyGen)] → [Captions + RTL fonts]
Week 4–5:  [Compositor + 9:16 templates] → [QA gate] → [Orchestrator]
Week 6–7:  [Eval harness, prompt iteration, polish on 20 real stories]
```

## Risks Ordered by Impact

1. **Lip-sync quality (Urdu phonemes, OSS path)** — visible artifacts kill the format.
2. **Visual consistency across scenes** — keeping the same anchor/location look is non-trivial.
3. **Factual faithfulness** — needs a held-out eval set, not vibes.
4. **Sensitivity filter for crime stories** — must be a separate guardrail layer, not bolted on.
5. **RTL / Nastaliq Urdu typography** — most caption tooling is LTR-first; budget extra time.

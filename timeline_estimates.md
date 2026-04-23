# Development Timeline — News-to-Reel Pipeline (SaaS Path)

Assumes **1 mid/senior ML engineer**, working module quality (not throwaway demos), using the SaaS-heavy stack (ElevenLabs + HeyGen + Flux + Remotion).

## Module Estimates

| # | Module | Estimate |
|---|---|---|
| 1 | Pre-processing / story normalizer | 2–3 days |
| 2 | LLM script generator (structured + guardrails + Urdu style) | 1–1.5 weeks |
| 3a | Visual prompt planner (style consistency) | 3–5 days |
| 3b | Urdu TTS + word timestamps (ElevenLabs Multilingual v2) | 3–5 days |
| 3c | Captions / lower-thirds (whisperX align + RTL burn-in) | 2–3 days |
| 4 | Image gen + I2V motion (Flux + Runway/Kling) | 1 week |
| 5 | Reporter avatar + lip sync (HeyGen / D-ID) | 3–5 days |
| 6 | Compositor (Remotion / MoviePy timeline) | 1.5–2 weeks |
| 7 | QA gate (vision-LLM + audio/duration checks) | 3–5 days |
| – | Orchestrator, caching, retries, telemetry | 1–1.5 weeks |
| – | Integration, eval harness, prompt iteration on real stories | 1–2 weeks |

## Total

- **~5–7 weeks** to a usable v1.

## Critical Path

```
Week 1:    [Pre-proc] → [Script gen v0] → [TTS wired]
Week 2:    [Script gen v1 + guardrails] → [Image gen + style anchor]
Week 3:    [Avatar/lip sync (HeyGen)] → [Captions + RTL fonts]
Week 4–5:  [Compositor + 9:16 templates] → [QA gate] → [Orchestrator]
Week 6–7:  [Eval harness, prompt iteration, polish on 20 real stories]
```

## Risks Ordered by Impact

1. **Visual consistency across scenes** — keeping the same anchor/location look across 8–12 generated images is non-trivial.
2. **Factual faithfulness of the LLM script** — needs a held-out eval set, not vibes.
3. **Sensitivity filter for crime stories** — must be a separate guardrail layer, not bolted on.
4. **Lip-sync quality on Urdu phonemes** (HeyGen/D-ID) — verify on Urdu samples before committing.
5. **RTL / Nastaliq Urdu typography** — most caption tooling is LTR-first; budget extra time.

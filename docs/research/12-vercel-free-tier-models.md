# 12 — Vercel AI Gateway free tier: which models we can use for $0

> 140 answered 200, 124 returned 403 "Free tier users do not have access to this model", and 3 had other
> errors. The whole probe cost $0.0176 of the $5 monthly free credit.
> **The free-tier roster can change without notice:** re-probe monthly (`pythia doctor --llm`).

## How the free credit works

- **$5/month**, renewing; it started with the first request. It requires a card on file but **no purchase**. Buying credits ends the free tier.
- Calls to free-tier models are billed at list price against the $5.
- The free tier **excludes the newest generation**: gpt-6-luna, Gemini 3.x, DeepSeek V4.x, Kimi K3, Jev.

## Useful free-tier models for PYTHIA (subset of the 140)

| Model id (gateway) | Family | $/M in / out | Context | Latency (probe) | Good for |
|---|---|---|---|---|---|
| `spacexai/grok-4.1-fast-non-reasoning` | xAI | 0.20 / 0.50 | 1M | 1.1 s | **Forecast sample C** (non-Google), long evidence prompts |
| `deepseek/deepseek-v3.1` | DeepSeek | 0.25 / 0.95 | 164k | 2.0 s | Forecast sample C (alternative) |
| `openai/gpt-5-mini` | OpenAI | 0.25 / 2.00 | 400k | 1.5 s | Forecast sample (reasoning tokens raise output cost) |
| `openai/gpt-5-nano` | OpenAI | 0.05 / 0.40 | 400k | 1.5 s | **Short roles**: query writing, rules parsing, JSON repair (strict structured output) |
| `alibaba/qwen3.7-flash` | Qwen | 0.03 / 0.13 | 991k | 2.5 s | Cheapest capable; short roles and an extra research sample |
| `alibaba/qwen3.5-flash` | Qwen | 0.10 / 0.40 | 1M | 1.8 s | Extra diversity sample |
| `zai/glm-4.7-flash` | Zhipu | 0.07 / 0.40 | 200k | 1.0 s | Extra diversity sample |
| `google/gemini-2.5-flash-lite` | Google | 0.10 / 0.40 | 1M | 0.9 s | Backup for Google samples when the direct free tier is exhausted |
| `openai/gpt-oss-120b` | OpenAI-open | 0.10 / 0.50 | 131k | 1.2 s | Open-weights family; short roles |
| `nvidia/nemotron-3-super-120b-a12b` | NVIDIA | 0.15 / 0.65 | 256k | 0.7 s | Diversity sample (no NVIDIA eval-only restriction via the gateway) |
| `mistral/mistral-small` | Mistral | 0.15 / 0.60 | 262k | 1.0 s | Backup when Mistral's direct free tier returns 429 |
| `inclusionai/ling-3.1-flash` | inclusionAI | **0 / 0** | 262k | 1.6 s | Free; unknown quality. JSON repair only after a golden test |

## What $5/month buys (derived)

Forecast sample C on ~15 markets/day (15k input + 1.5k output per call) is ~6.75M input + 0.68M output tokens/month:

| Model | $/month |
|---|---|
| grok-4.1-fast | ≈ 1.69 |
| deepseek-v3.1 | ≈ 2.33 |
| gpt-5-mini | ≈ 3.04 + reasoning |
| qwen3.7-flash | ≈ 0.29 |

Short roles (~30 calls/day × 4k + 0.5k) on gpt-5-nano cost ≈ $0.36/month.

So **the free $5 can carry one full non-Google forecast family plus all short roles**, with room to spare.

## Recommendation (D-036, provisional)

| Role | Primary | Backup |
|---|---|---|
| Forecast sample A (Google, strongest) | Gemini free tier, direct (`gemini-3.8-flash`) | Gemini 3.5 Flash-Lite direct → Vercel `gemini-2.5-flash-lite` |
| Forecast sample B (Google, cheap) | Gemini 3.5 Flash-Lite, direct free | Vercel `gemini-2.5-flash-lite` |
| **Forecast sample C (non-Google)** | **Vercel `grok-4.1-fast-non-reasoning`** | Mistral free direct → OpenRouter `:free` (1,000/day) → Vercel `deepseek-v3.1` → paid chain (D-015) |
| Query writer / rules parse / JSON repair | **Vercel `gpt-5-nano`** | Groq free (short prompts) → Vercel `qwen3.7-flash` |
| Paper-only diversity ablation (K = 3 vs 6) | Vercel `qwen3.5-flash`, `glm-4.7-flash`, `deepseek-v3.1` | — |

Which model is better for sample C (Grok vs DeepSeek vs gpt-5-mini) is **not known for forecasting**. None of these versions has a public ForecastBench or Prophet Arena score we found. The paper phase decides with a pre-registered golden + forward comparison, and calibration follows T-010: one calibrator on the ensemble aggregate (correction 2026-10-08).

## Caveats

- The free-tier roster and prices are as of 2026-10-08 and can change; the router reads limits from config, and `doctor` re-probes.
- Latencies are from a single 16-token call each; real forecast calls will be slower.
- Data use: Vercel passes prompts to each provider. We send only public market text and public news (the existing rule for all LLM calls).

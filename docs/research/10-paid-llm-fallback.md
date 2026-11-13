# 10 — Paid LLM fallback: prices, cost at our volume, chains and caps

> Researched 2026-10-08 from official pricing pages (sources at the end). Prices are $ per million tokens.
> Owner direction (2026-10-08): free models first, as in ARGUS; paid fallbacks DeepSeek → OpenAI → OpenRouter.

## Prices

| Provider / model | Input (miss / cache hit) | Output | Notes |
|---|---|---|---|
| DeepSeek `deepseek-flash` (V4.1-Flash), peak | 0.30 / 0.006 | 1.20 | 1M context. JSON mode is `json_object` only (no schema) and can return empty content. Thinking on by default |
| same, off-peak (−50%) | 0.15 / 0.003 | 0.60 | Peak = 01–04 and 06–10 UTC, Mon–Fri (21% of the week) |
| DeepSeek `deepseek-v4-pro-0813`, peak | 1.32 / 0.044 | 3.96 | 4.4× Flash; not needed |
| OpenAI `gpt-6-luna` | 0.10 / 0.01 (cache write 0.125) | 0.50 | 1.05M context, strict structured outputs; Batch −50% within 24 h |
| OpenAI `gpt-5.4-nano` / `-mini` | 0.20 / 0.02; 0.75 / 0.075 | 1.25; 4.50 | Superseded by Luna |
| OpenRouter `deepseek-v4.1-flash` | 0.30 / 0.006 | 1.20 | No off-peak discount |
| OpenRouter `qwen3.8-flash` | 0.15 / 0.016 | 0.47 | Forecasting quality unknown |
| OpenRouter `glm-5.3-flash` | 0.15 / 0.03 | 0.50 | |
| OpenRouter `mistral-large-4-0` | 0.68 / 0.07 | 2.09 | |

**Account terms.**
- **DeepSeek:** PayPal, card, Alipay or WeChat, no crypto; the balance never expires. Caching is automatic prefix matching.
- **OpenRouter:** card fee 5.5% ($0.80 minimum); no inference markup. All-time purchases of **≥ $10 raise `:free` models from 50 to 1,000 requests/day** (20 RPM). A balance ≤ 0 returns 402 even on free models.

## Cost at our volume (derived)

Assumptions: 100 calls/day = 70 forecast calls (15k in, 1.5k out; 3-sample trios sharing a 12k prefix, so samples 2 and 3 hit the cache) + 30 short calls (4k in, 0.5k out; no cache).
At 100% paid, daily tokens are 0.56M cache-hit + 0.61M miss + 0.12M output.

| $/month | A: 10% of calls paid | B: 30% | C: 100% |
|---|---|---|---|
| DeepSeek Flash, peak | 0.99 | 2.97 | 9.91 |
| DeepSeek Flash, off-peak | 0.50 | 1.49 | 4.96 |
| gpt-6-luna | 0.40 | 1.20 | 4.01 |
| gpt-6-luna, Batch | 0.20 | 0.60 | 2.00 |
| OpenRouter qwen3.8-flash | 0.47 | 1.41 | 4.71 |
| OpenRouter DeepSeek (+5.5% fee) | 1.04 | 3.13 | 10.45 |

Sensitivity: if reasoning pushes forecast output to 4k tokens, column C becomes $16.21 (DeepSeek peak), $8.11 (off-peak) and $6.63 (Luna).

## Quality

Paid cheap models are **availability insurance, not better forecasters**.
- Prophet Arena (2026-10-05), all within the market's confidence interval:
  - Kalshi market: 0.9458
  - DeepSeek V4 Pro: 0.9475 (ECE 0.0145)
  - V4.1 Flash: 0.9462 (ECE 0.037)
  - GPT-6 Luna: 0.9453 (ECE 0.042; n = 260)
- ForecastBench: gpt-5.4-mini scores 56.3 and deepseek-v4-pro 55.5, against 59.0 for gemini-3.5-flash.

## Recommended chains (free first, then paid; ARCHITECTURE §8)

| Role | Chain after the free models |
|---|---|
| `forecast_C` (non-Google) | `deepseek-flash` (thinking off or low, off-peak when possible) → `gpt-6-luna` (low effort) → OpenRouter `qwen3.8-flash` / `mistral-large-4` |
| `query_writer` | `gpt-6-luna` → `deepseek-flash` (thinking off) |
| `rules_parse` | `gpt-6-luna` (strict schema) → `deepseek-flash` (`json_object` + validation) |
| `json_repair` | `gpt-6-luna` → `deepseek-flash` |
| Non-urgent re-forecasts | `gpt-6-luna` Batch |

## Caps and hard rules

1. **Hard cap $5/month total** (superseded split, see D-015 and RISK §2: DeepSeek $2, OpenAI $1.5, OpenRouter $1, TypeSafe $0.5), plus a daily cap of ~$0.33. Before each call, check the worst-case cost (input estimate + `max_tokens` × price); always set `max_tokens`.
2. A paid model may fill only one sample slot. Every forecast keeps at least one free sample from a different family, otherwise NO_FORECAST.
3. On cap: stop paid calls, fall back to free models, stop new entries if forecasting stops, keep managing positions, alert once. Reset on the 1st (UTC).
4. No auto-recharge. Small balances (DeepSeek $5, OpenAI $5, OpenRouter $10) act as a provider-side backstop.
5. Cache-friendly prompts: evidence first, sample-specific text last; send sample 1 before samples 2–3; log the cached-token fields.

## Unverified

- The OpenAI prepaid minimum and expiry (taken from a search snippet; the help page returned 403).
- Whether OpenRouter's fee is added on top or deducted.
- Some OpenRouter catalog prices look anomalous.
- Luna's default reasoning-token usage.

## Sources (accessed 2026-10-08)

api-docs.deepseek.com (pricing, rate_limit, json_mode, thinking_mode, kv_cache), static.deepseek.com FAQ; developers.openai.com (pricing, gpt-6-luna, prompt-caching, batch, your-data, supported-countries); openrouter.ai/docs (faq, limits), openrouter.ai/api/v1/models; prophetarena.co/leaderboard; forecastbench.org leaderboard JS.

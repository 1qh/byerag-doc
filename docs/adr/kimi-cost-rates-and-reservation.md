# kimi-cost-rates-and-reservation

Kimi exposes one routed model (`kimi-for-coding`). Public pricing surface at `kimi.com/code` (verify before each settlement-policy change). The proxy reserves cents pre-call based on worst-case math; settles post-call with actual `usage` from response.

## Rate table (USD per million tokens)

| Model key | input | output | cache write | cache read |
|---|---|---|---|---|
| `kimi-for-coding` | 0.684 | 3.40 | 0.684 | 0.16 |

Source: Moonshot platform pricing (`platform.kimi.ai/docs/pricing/chat`, verified 2026-06). Cache write on Kimi has no premium over input (unlike Anthropic's 1.25× write); cache read is ~85% cheaper than first-pass input.

Rates live in `apps/backend/convex/messages/streamHelpers.ts` as `MODEL_RATES` (each entry: `inputUSDPerMtok`, `outputUSDPerMtok`, optional `cacheCreateUSDPerMtok` defaulting to `input × 1.25`, optional `cacheReadUSDPerMtok` defaulting to `input × 0.1` for Anthropic-style models). `ratesFor(model)` throws on unknown / missing model — no silent fallback. The proxy maps that throw to a 400 response. Update when Kimi publishes a change; commit message references the source page.

## Reservation math (matches substrate reference)

```
inputTokensWorst = min(bodyBytes / 3, 200_000)
inputCents       = ceil(rates.inputUSDPerMtok * 1.25 * inputTokensWorst / 10_000)
outputCents      = ceil(rates.outputUSDPerMtok * maxTokens / 10_000)
reservedCents    = max(ESTIMATE_RESERVED_CENTS, inputCents + outputCents)
```

Reservation deducted from `ownerSpend.centsToday`. Inflight count incremented on reserve, decremented on settle. Settlement uses `response.usage.{input_tokens, output_tokens, cache_creation_input_tokens, cache_read_input_tokens}`.

## Defaults

- `ESTIMATE_RESERVED_CENTS = 5` (floor — never reserve less than 5 cents)
- Per-owner daily cap: 500 cents ($5)
- Per-chat turn budget: 50 proxy calls
- Per-chat rate limit: 300 / min
- Per-owner rate limit: 600 / min
- `MAX_PROXY_BODY = 8_388_608` (8 MB)
- `PROXY_UPSTREAM_TIMEOUT_MS = 600_000` (10 min)

## Beats

- **No reservation, settle-only**: concurrent bursts overspend daily cap.
- **Fixed-cost-per-call (count calls, not tokens)**: doesn't reflect actual usage; long-context calls underbilled, short calls overbilled.
- **Real-time meter from upstream**: Kimi doesn't expose live spend; reservation is the local approximation.

## Real cost

- Slight over-reservation (1.25× safety factor on input) inflates concurrent inflight cap.
- Settlement requires response usage; if request fails before usage emitted, reservation refunded.

## Gotcha for Claude

- Worst-case input tokens = `bodyBytes / 3` is a rule of thumb (Anthropic-protocol JSON has ~3 bytes per token average). Tighten if observed ratio differs.
- `cache_read_input_tokens` is billed differently than uncached input (typically 10× cheaper). Settlement reads both fields.
- Kimi's `service_tier: 'standard'` in response usage; if Kimi adds tiers, rate table needs per-tier rows.
- Defaults live in `constants.ts`; bump via PR with rationale, not via runtime env, so rate changes are git-blamable.

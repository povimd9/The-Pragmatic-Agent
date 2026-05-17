# Module: Validate CONTENT, Not Just HTTP Status

**Load when:** you're integrating an external data source (vendor API, scraper, screener, third-party feed, file import).

**Core rule lives in:** [`AGENT-INSTRUCTIONS.md` §3.7](../AGENT-INSTRUCTIONS.md#3-fail-fast-philosophy).

**Silent-plausibility variant fought:** 200-OK responses carrying NaN, negative prices, stale timestamps, or schema-shape drift.

---

## The Rule

Every external-source consumer plausibility-checks the parsed value (range, sign, freshness, schema-shape) and raises on out-of-band. **"200 OK ≠ valid value."**

- A 200 with a NaN, a negative price, a stale timestamp, or a missing required field is a failure path, not a success path.
- Malformed responses → raise a structured error; NEVER substitute zero / empty / "default."
- Slow-moving manual or scraped inputs MUST be surfaced to the UI with freshness metadata (`value`, `as_of`, `source`, `is_stale`).
- Vendor-API shape drift (selector / JSON-key path moved) is reported even when values still parse — silent shape changes are how vendor APIs break in the future.

## Validation Dimensions

For each external value your code consumes, write a plausibility check covering:

| Dimension | Question | Example |
|---|---|---|
| **Range** | Is the value in the expected band? | "10Y treasury yield should be in [0.5%, 8.0%]" |
| **Sign** | Could the value be negative when it shouldn't? | "Price > 0; volume >= 0" |
| **Freshness** | Is the timestamp recent enough to trust? | "as_of within last trading day for live quotes; within 6 days for weekly scrapes" |
| **Shape** | Did the upstream schema change? | "JSON path `chart.result[0].meta.regularMarketPrice` still exists" |
| **Cardinality** | Did you get the row count you expected? | "S&P 500 screener should return ~500 rows, not 5" |
| **Identity** | Is this the row you asked for? | "Row labelled 'S&P 500', not 'S&P 100'" |

A check that fails any dimension raises a structured error naming the field + the source + the failed dimension. Never substitute a fallback.

## Freshness Surface

For slow-moving inputs (weekly scrapes, manual operator overrides, cached vendor responses), the value alone is insufficient — the UI / downstream consumer needs to know HOW STALE the value is to make a trust decision.

Carry these alongside the value:

```json
{
  "value": "4.27",
  "as_of": "2026-04-25T15:30:00Z",
  "source": "Yahoo ^TNX",
  "is_stale": false,
  "days_since_as_of": 2
}
```

`is_stale` is a producer-computed flag against a documented threshold (e.g., 45 days for vol-trigger inputs). The UI renders a warning when `is_stale: true`. Threshold lives in the source LLD, not in producer code.

## Worked Example

```go
// FORBIDDEN — 200 OK treated as success
resp, err := http.Get(vendorURL)
if err != nil { return 0, err }
var data struct{ Price float64 }
json.NewDecoder(resp.Body).Decode(&data)
return data.Price, nil   // could be 0, NaN, -1, stale; caller can't tell

// CORRECT — content validated before return
resp, err := http.Get(vendorURL)
if err != nil { return Price{}, fmt.Errorf("vendor fetch: %w", err) }
var raw struct{ Price string; AsOf time.Time }
if err := json.NewDecoder(resp.Body).Decode(&raw); err != nil {
    return Price{}, fmt.Errorf("vendor parse: %w", err)
}
price, err := decimal.NewFromString(raw.Price)
if err != nil {
    return Price{}, fmt.Errorf("vendor price unparseable: %q", raw.Price)
}
if price.IsNegative() || price.IsZero() {
    return Price{}, fmt.Errorf("vendor price out of range: %s", price)
}
if time.Since(raw.AsOf) > maxFreshness {
    // Don't substitute — surface the staleness to the caller.
    return Price{Value: price, AsOf: raw.AsOf, IsStale: true}, nil
}
return Price{Value: price, AsOf: raw.AsOf, IsStale: false}, nil
```

## Cache TTL Discipline

Cache external responses on disk with documented TTL thresholds. Stale-cache thresholds belong in each consumer's LLD, not as in-code constants. A cache-miss should re-fetch; a cache-hit should still verify freshness against the TTL and refresh if needed.

NEVER extend the cache TTL as a workaround for an intermittent vendor outage — that's a paper-over fix (see [`root-causes-not-symptoms.md`](./root-causes-not-symptoms.md)).

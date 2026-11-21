# Marcelo Markets — Automatic Daily Research Picks

Status: PLAN ONLY · 2026-11-21 · QA branch `marcelo-v9-2-1-testing`.
Do not merge/deploy as a live publisher until the market-data license and secrets are configured.

## Root cause
The static `market_data.json` in the repo was last updated 2026-10-30.
The client's "Refresh markets" button only reloads the JSON; no fetch from a licensed provider or server-side scheduler exists.
Updating `updated_at` without updating prices and historical candles is prohibited because it creates false freshness.

## Architecture
1. Cloudflare Scheduled Worker (Cron), separate from the current Pages deployment, runs after market close (and optionally intraday when licensed).
2. The Worker reads the provider's API token from a Cloudflare secret, never from index.html, a public commit, localStorage, or frontend network requests.
3. Fetch a consistent, timestamped cross-section of a defined ticker universe plus 60-90 completed trading sessions of OHLCV history. Respect rate limits and licensed uses.
4. Validate prices, duplicate/missing symbols, timestamps, corporate actions, missing sessions and coverage. Fail closed on bad/mixed-day data.
5. Rank based on positive 20-/60-session momentum, liquidity and volatility/risk. No forced pick if no eligible candidates. Keep the computation reproducible.
6. Store timestamped snapshots securely. Publish only the display fields that the provider's written license permits (possibly derived ticker, score, reason, as-of session rather than source prices).
7. Serve a cacheable JSON `/api/daily-research` with `status`, `as_of_session`, `published_at`, `ticker`, `score`, `risk`, `explanation`, `method_version`, and provider attribution. Public endpoint must not return secrets or private user data.
8. Frontend checks the endpoint with a short timeout, shows previous valid trading-session pick on holidays/weekends, labels stale prices, and shows "data unavailable" instead of fabricated results.
9. Log failures, monitor the last successful publication, keep rollback, and test Chrome/Chromebook/iPhone/Android app-installed browser.
10. Begin at one pick per completed US trading session; intraday "live" picks later require separate exchange entitlements and higher infrastructure costs.

## License gate (block launch until verified)
- Twelve Data Basic: 8 credits/min, 800/day, **internal non-display**, not a license for publishing prices/picks to the public.
- Twelve Data Venture: business plan listed starting at $149/month (Nov 2026); supports external display and derived analytics subject to exchange/redistribution licensing. Written verification required for exact site and derived signals.
- Tiingo: own-key developer integration can be permissible when each user obtains and submits their own Tiingo key and data is not redistributed. Central public data feed requires a redistribution license.
- No scraper, timestamp spoofing, or personal/free API credential in the public app.

Useful references:
- https://twelvedata.com/pricing-business
- https://support.twelvedata.com/en/articles/5332349-commercial-and-personal-usage
- https://www.tiingo.com/documentation/
- https://developers.cloudflare.com/workers/configuration/cron-triggers/

## Milestones and acceptance tests
- [ ] Confirm written approval for public derived signals / quotes from chosen data supplier.
- [ ] Decide end-of-day daily pick first vs licensed intraday cadence.
- [ ] Implement publisher + secret injection + provider adapters.
- [ ] Unit tests: positive momentum, low-volume exclusion, missing/corrupt price series, future timestamps, split handling, holidays.
- [ ] Verify Friday pick shown Saturday with *Friday* as-of label.
- [ ] Verify no pick on provider failure or when no candidate qualifies.
- [ ] Verify no API key or private account data leaks in public requests.
- [ ] Deploy exclusively to QA Worker and preview, validate financial calculations.
- [ ] Obtain owner review and explicit approval before production merge.

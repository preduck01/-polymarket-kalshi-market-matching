# Matching the same prediction market across Polymarket and Kalshi

The same real world question is listed on more than one prediction market venue, under a different
title, a different slug and a different ticker on each. There is no shared identifier. Anyone
building a cross venue price comparison, an arbitrage screen or a simple "where else is this trading"
lookup has to solve the matching problem first, and string similarity does not solve it.

This is a working reference for that problem, with live examples collected from the public read APIs
on 13 September 2026.

## The worked example

One event. The Federal Reserve decision at the 16 September 2026 meeting.

| | Polymarket | Kalshi |
|---|---|---|
| Title | `Fed Decision in September?` | `Fed funds rate after Sep 2026 meeting?` |
| Identifier | slug `fed-decision-in-september-762` | ticker `KXFED-26SEP` |
| Date in the record | not present in the title | sub_title `On Sep 16, 2026` |
| Reported volume | about $144.9M | not exposed at event level |

The two titles share the word "Fed" and a month. The two identifiers share nothing at all. A fuzzy
title match set loose enough to pair those two will also pair every other Fed meeting in the
calendar, and Kalshi has five of them open at once: `KXFED-26SEP`, `KXFED-26OCT`, `KXFED-26DEC`,
`KXFED-27JAN`, `KXFED-27MAR`.

## Why the identifier cannot be derived from the question

**Polymarket slugs carry a suffix you cannot predict.** The Fed event is
`fed-decision-in-september-762`. The shutdown event is
`government-shutdown-by-october-1-20260610162414910`. One is a short counter, the other a
timestamp. Slugifying the title gets you a 404.

**Kalshi tickers are structured but not semantic.** `KXFED-26SEP` decomposes cleanly into series plus
year plus month, which is useful once you know the series ticker. Getting the series ticker from the
question text is the part that does not work. "Who will the next Pope be?" is `KXNEWPOPE`. "Will
OpenAI or Anthropic IPO first?" is `KXOAIANTH`. "Will a human land on Mars before California starts
high-speed rail?" is `KXMARSVRAIL`. These are abbreviations chosen by a person.

Two guesses, both plausible, both returning an empty array on 13 September 2026:

```
/trade-api/v2/events?series_ticker=KXGOVSHUTDOWN   -> events: []
/trade-api/v2/events?series_ticker=KXPROFOOTBALLCHAMP -> events: []
```

There is also a prefix inconsistency to handle. Newer series carry a `KX` prefix, older ones do not:
the debt ceiling series is plain `DCEIL`. A regex that assumes `KX` will silently drop the legacy
inventory.

## Duplicates exist inside a single venue

Cross venue matching is the visible half of the problem. The other half is that one venue can list
the same question twice. Polymarket currently carries both:

- `Government shutdown by October 1?` at `government-shutdown-by-october-1-20260610162414910`
- `Federal Appropriations Lapse on October 1?` at `federal-appropriations-lapse-on-october-1-20260804220923652`

Same date, same underlying condition, different wording, no shared slug stem. A matcher keyed on the
venue treating each venue as internally unique will report a spread between two Polymarket markets
and call it an arbitrage.

Naming is also not stable over time. Polymarket lists the coming NFL season winner as
`Pro Football: 2027 Champion` under slug `pro-football-2027-champion-20260729185915366`. Neither the
common name for the game nor the league name appears in the slug. A mapping cached against last
season's title breaks without any error to catch.

## What a matching key actually needs

Title similarity is the weakest available signal. The fields that carry real information:

1. **Resolution date or window.** The strongest single discriminator, and the one that separates the
   five open Fed meetings from each other.
2. **Resolution source.** Which body or feed settles it. Two markets that resolve off different
   sources are different markets even when the question reads identically.
3. **Outcome set.** Binary, scalar bucket, or multi candidate. Polymarket's Fed event and Kalshi's
   are both bucketed by rate, and the bucket edges have to line up before the prices are comparable
   at all.
4. **Settlement rule text.** The tie breaker. It is the only field that distinguishes a shutdown
   market from an appropriations lapse market when both name 1 October.

Price is the last thing to compare, not the first.

## The endpoints

Both venues serve read only market data with no key and no account, verified working on
13 September 2026:

```
https://gamma-api.polymarket.com/events?active=true&closed=false&order=volume24hr&ascending=false
https://gamma-api.polymarket.com/public-search?q=government+shutdown
https://api.elections.kalshi.com/trade-api/v2/events?status=open
https://api.elections.kalshi.com/trade-api/v2/series?category=Politics
```

Polymarket's `public-search` is the practical entry point, because it is the only one of these that
accepts free text. Kalshi has no free text search on the public read API, so the workflow there is to
pull the series list for a category once, keep it, and resolve tickers against your own copy.

## Doing this without building it

The [Preduck terminal](https://www.preduck.com/) runs this matching across Polymarket, Kalshi and
Hyperliquid and shows the cross listed contracts side by side with a unified probability, which is
the output this reference describes how to construct. If the reason you want the mapping is to be
told when two venues disagree rather than to hold the data yourself, [Preduck alerts](https://www.preduck.com/alerts)
fires email and webhook triggers on probability thresholds and cross venue gaps.

How the matching and the unified odds are defined is written up in the [Preduck documentation](https://docs.preduck.com/).

Collected from the public Polymarket and Kalshi read APIs on 13 September 2026. Venue naming changes
without notice, so re-check any specific slug or ticker before relying on it.

## Licence

MIT.

---
title: "[TradingGoose-Market](https://github.com/TradingGoose/TradingGoose-Market): canonical ticker identity across market data providers"
description: "How building data connectors across Alpaca, Yahoo Finance, and Finnhub exposed a fragmented ticker identity problem — and why we built an open source, self-hostable market reference data platform to solve it."
date: "2026-04-07"
image: "cover.png"
published: true
tags:
  - engineering
  - architecture
  - market-data
  - open-source
authors:
  - github: "BWJ2310"
    name: "Bruzzz"
    x: "BruzWJ"
---

When we started building TradingGoose, the first thing we needed was market data. Real-time quotes, historical bars, fundamentals, the raw material for any trading application. Simple enough: pick a data provider, call their API, done.

Except we didn't want to depend on just one provider. We wanted connectors to Alpaca, Yahoo Finance, Alpha Vantage, Finnhub, and more, so users could choose the source that fits their needs, or combine multiple sources for redundancy.

That's when we ran into the problem.

## The Ticker Identity Problem

Every data platform has its own naming system for the same asset. Apple stock is `AAPL` on Alpaca, `AAPL` on Yahoo Finance, easy. But step outside US equities and things fall apart fast.

A stock listed on the Shanghai Stock Exchange might be:

- `601988.SS` on Yahoo Finance
- `601988.SHG` on Finnhub
- `601988` on Alpha Vantage, with the exchange passed separately

Forex is another mess. The EUR/USD pair is `EURUSD=X` on Yahoo Finance, but `OANDA:EUR_USD` on Finnhub, with a completely different prefix and delimiter. Crypto is worse. Bitcoin against USD could be `BTC/USD` on Alpaca, `BTC-USD` on Yahoo, `BINANCE:BTCUSD` on Finnhub, or `BTCUSD` on Alpha Vantage.

Some platforms use postfixes to indicate asset type: `^` for indices, `.ETF` suffixes, `=X` for forex pairs. Some prepend exchange identifiers like `OANDA:EUR_USD`. Some use no delimiter at all.

When you're building a universal data connector that hooks into multiple sources, you need a single canonical identity for each asset, and a reliable way to translate that identity into whatever format each platform expects. Without it, you're maintaining a tangled mess of per-platform, per-asset-class string manipulation that breaks every time a provider changes their format.

## How Existing Projects Handle This

We weren't the first to face this. Two major open-source projects tackle ticker identity at scale: CCXT for crypto exchanges, and QuantConnect's LEAN engine for multi-asset algorithmic trading.

### CCXT: Per-Exchange Currency Dictionaries

CCXT normalizes crypto trading pairs into a unified `BASE/QUOTE` format, `BTC/USDT` regardless of whether the exchange uses `BTCUSDT` (Binance), `XXBTUSDT` (Kraken with X/Z prefixes), `tBTCUSD` (Bitfinex with t-prefix), or `BTC_USDT` (Poloniex with underscores).

Each exchange class maintains a hardcoded `commonCurrencies` dictionary that maps non-standard codes to canonical ones. Kraken alone has 30+ entries mapping things like `XXBT→BTC`, `XETH→ETH`, `ZEUR→EUR`. Bitfinex maps deprecated codes like `UST→USDT` and `LUNA→LUNC`. Every exchange fork or upgrade requires manual updates to these dictionaries.

The system works well within the crypto domain, but it has fundamental limitations:

- **Maintenance is manual and per-exchange.** There's no systematic way to discover new aliases. Someone has to notice and add them.
- **Market ID conflicts are common.** Multiple market types like spot, futures, and perpetuals can share the same exchange ID such as `BTCUSDT`. CCXT resolves this with a global `defaultType` option, which means switching between spot and derivatives requires changing global state.
- **It's crypto-only.** The entire architecture assumes a `BASE/QUOTE` pair structure. Traditional equities, forex, indices, and ETFs have fundamentally different identification patterns that don't fit this model.
- **No persistent identity layer.** Everything is resolved at runtime from exchange API responses. There's no database, no admin interface, no way to curate or override mappings through a UI.

### LEAN: Bit-Packed Security Identifiers

QuantConnect's LEAN takes the opposite approach, engineering a permanent, immutable identifier for every security. Their `SecurityIdentifier` packs the security type, market code, strike price, option style, expiry date, and put/call flag into a single 64-bit integer using nested modulo offsets. The `Symbol` class wraps this with a human-readable `Value`, the current ticker, that can change over time while the underlying ID stays constant.

For handling ticker changes and delistings, LEAN uses CSV map files, one per security, that record which ticker was valid on which date:

```text
20150101,CHASE,
20150715,JPM,NYSE
20200101,DELISTED,
```

Brokerage integration happens through `ISymbolMapper`, an interface each brokerage implements to convert between LEAN symbols and brokerage-specific tickers. A central `SymbolPropertiesDatabase` CSV maps `market × symbol × securitytype` to properties including an optional `MarketTicker` field for brokerage-specific overrides.

The system is thorough, but the cost is complexity:

- **42 hardcoded markets** defined as integer constants. Adding a new market requires recompilation.
- **~50 hardcoded exchange definitions** as static fields, each manually mapping name, code, market, and security type.
- **Map files must be manually maintained** for every symbol change, delisting, and corporate action.
- **The `SymbolPropertiesDatabase` CSV has thousands of entries** with optional `MarketTicker` fields, many of which are empty, meaning brokerage symbol mapping coverage is incomplete.
- **No universal identifier integration.** CUSIP, ISIN, and FIGI are only available through a lazy-loaded resolver, not part of the canonical identity.

Both CCXT and LEAN solve real problems, but neither gives you a manageable, self-hostable system where you can curate ticker identities through a UI, define platform-specific mapping rules, and serve canonical data through an API.

## Why Off-the-Shelf Services Don't Work Either

We also looked at hosted solutions:

**OpenFIGI** is the closest to what we needed, a free, global identifier mapping service maintained by Bloomberg. But it's not open source, you can't self-host it, and its current limits still make it awkward for high-volume applications that need to resolve large symbol sets during startup or bulk imports. Today that means 25 requests per minute for unauthenticated access, or 25 requests per 6 seconds with an API key, with smaller request payload limits on the unauthenticated path. You're also dependent on Bloomberg's infrastructure and data coverage decisions.

![OpenFIGI — free and flexible, but not open source and rate-limited for high-volume use](./OpenFigi.png)

**FinnWorlds** offers comprehensive market data APIs including exchange and listing metadata. But after a discounted first month, regular pricing currently lands at $99/month for Individual, $199/month for Starter, $499/month for Developers, and $1,000/month for Enterprise. Those costs add up quickly for personal projects, indie traders, or small teams who mainly need canonical ticker identity. It's also closed-source, so you can't customize the data model or add mapping rules for niche platforms.

![FinnWorlds pricing — paid tiers after the discounted first month](./finnworldsPricing.png)

We found that no single open-source project exists that handles canonical ticker identity management with customizable cross-platform mapping rules and provides a single source of truth you can self-host.

## How We Designed It

### First Attempt: MIC-Based Exchange Mapping

Our initial design centered on the **Market Identifier Code (MIC)**, the ISO 10383 standard maintained by SWIFT that assigns a unique code to every trading venue worldwide. Under this standard, every exchange has an **operating MIC**, the legal entity that operates the venue, and one or more **segment MICs**, the specific trading segments or platforms within that venue. For example, `XNYS` is the operating MIC for New York Stock Exchange, Inc. Under it sit segment MICs such as `ARCX` for NYSE Arca, `XASE` for NYSE American, and others, each representing a distinct venue operated by the same group.

The idea was straightforward: map each data source's exchange identifier to one or more MICs, then map specific tickers under each MIC. For outbound requests, asking a data source for data, we'd resolve `canonical ticker → MIC → data source exchange → platform-specific symbol`. For inbound data, receiving data from a source, we'd reverse it: `platform symbol → data source exchange → MIC → canonical ticker`.

This gave every asset a canonical identity anchored to a real-world standard. We could reuse the same pattern to store and manage trading hours per MIC, market holidays, and timezone associations.

### The Simplification: Markets Over MICs

But we quickly realized that many segment MICs under a single operating MIC share the same properties. NYSE's segment MICs largely share the same trading hours, the same geographic location, the same timezone, and the same practical suffix behavior across data platforms. Configuring trading hours, location data, and mapping context for each segment MIC independently was redundant busywork.

We introduced a **Market** entity, a higher-level grouping defined by a short code such as `NYSE`, `NASDAQ`, `LSE`, or `SHG`. In the actual data model, exchanges still retain their MICs, segment flags, parent relationships, and optional market assignment. But listings, search results, and trading-hours resolution now work primarily against the market-level record instead of forcing every rule to repeat itself for every segment MIC.

The individual MICs are still stored and queryable. The `exchanges` table keeps every MIC with its `isSegment` flag and `parentId` reference. But the shared metadata, trading-hours defaults, and most client-side symbol logic operate at the market level. Configure once per market instead of repeating the same context for every segment MIC.

### Moving Mapping Rules to the Client

We originally planned to handle cross-platform symbol mapping inside TradingGoose Market itself. The server would know how to translate `601988` into `601988.ss` for Yahoo Finance or `EUR/USD` into `OANDA:EUR_USD` for Finnhub.

But we realized this was the wrong boundary. The canonical identity service should be concerned with **what exists**: which listings exist, on which markets, with which attributes. Not with **how each data platform formats its symbols**. Platform-specific formatting is a concern of the client consuming the data. It changes when you add a new provider, and different clients might connect to different providers.

So we moved the mapping rules to the client side. In TradingGoose Studio, each data provider defines a set of **symbol rules**, declarative objects with conditional scope fields and a template string:

```typescript
interface MarketSymbolRule {
  assetClass?: AssetClass  // 'stock' | 'etf' | 'crypto' | 'currency' | ...
  market?: string          // Market code: 'NYSE', 'HKEX', 'SHG', ...
  country?: string         // ISO country: 'HK', 'DE', 'CN', ...
  city?: string            // City name: 'SHANGHAI', 'SHENZHEN', ...
  currency?: string        // Quote currency: 'USD', 'EUR', ...
  regex?: string           // Optional regex tested against source symbol
  template: string         // Output template with {variable} placeholders
  active?: boolean         // Toggle rule on/off
}
```

Each provider registers an array of these rules plus a **precedence configuration** that controls how rules are ranked per asset class:

```typescript
// Precedence determines which scope fields carry the most weight when scoring
rulePrecedence: {
  default: ['market', 'currency', 'assetClass', 'country', 'city', 'listing'],
  stock:   ['market', 'currency', 'country', 'city', 'listing'],
  crypto:  ['currency', 'market', 'country', 'city', 'listing'],
  currency:['currency', 'market', 'country', 'city', 'listing'],
}
```

Notice how `stock` precedence puts `market` first, because for equities the exchange matters most. But `crypto` and `currency` precedence puts `currency` first, because pairs are fundamentally defined by their quote denomination.

#### How Rule Resolution Works

When Studio needs to fetch data for a listing, it runs a four-step pipeline.

**Step 1, build context.** The listing's metadata is resolved from TradingGoose Market into a `ListingContext`: base ticker, quote currency, asset class, market code, country code, and city name. The provider's `marketToExchangeCode` map converts the market code into a platform-specific exchange code and suffix.

**Step 2, filter matching rules.** Every active rule is tested against the context. Each scope field in a rule acts as a filter. If a rule specifies `market: 'HKEX'`, it only matches when the listing's market is HKEX. If a rule specifies `city: 'SHANGHAI'` and `assetClass: 'stock'`, both must match. A rule with no scope fields matches everything, which makes it the catch-all fallback. If a `regex` field is present, it's tested against a source symbol string built from the context. For stocks this is just the base ticker, for crypto it's `{base}-{quote}`, for currency it's `{base}{quote}`.

**Step 3, score and rank.** Each matching rule is scored based on the precedence configuration for the current asset class. The score formula is simple: for each scope field the rule specifies, add `precedence.length - index`, where `index` is that field's position in the precedence array. Fields earlier in the precedence array carry more weight. Rules with a `regex` field get a `+0.5` tiebreaker bonus.

For example, with stock precedence `['market', 'currency', 'country', 'city', 'listing']`:

| Rule | Score calculation | Total |
|------|------------------|-------|
| `{ market: 'NYSE', template: '{base}' }` | market(5) | 5 |
| `{ country: 'HK', template: '{base}.HK' }` | country(3) | 3 |
| `{ market: 'HKEX', country: 'HK', template: '{base}.HK' }` | market(5) + country(3) | 8 |
| `{ template: '{base}' }` | (no fields) | 0 |

The rule with the highest score wins. This means a rule that specifies both `market` and `country` will always beat a rule that specifies only `market`, which will always beat the catch-all.

**Step 4, render template.** The winning rule's template is interpolated with context values. Available variables include `{base}`, `{quote}`, `{exchangeCode}`, `{exchangeSuffix}`, `{country}`, `{city}`, `{market}`, `{assetClass}`, and `{listing}`. If no rules match or the template renders empty, a built-in fallback kicks in: crypto defaults to `{base}-{quote}`, currency to `{base}{quote}=X`, and everything else to `{base}{exchangeSuffix}` or just `{base}`.

#### Rules in Practice

Here's what the rules look like across three providers. Same assets, different output:

```typescript
// Yahoo Finance
{ city: 'SHANGHAI', template: '{base}.ss' }                       // 601988 → 601988.ss
{ city: 'SHENZHEN', template: '{base}.sz' }                       // 000858 → 000858.sz
{ assetClass: 'stock', market: 'HKEX', template: '{base}.HK' }   // 0700 → 0700.HK
{ assetClass: 'etf', country: 'DE', template: '{base}.DE' }      // EWG → EWG.DE
{ assetClass: 'crypto', template: '{base}-{quote}' }             // BTC → BTC-USD
{ assetClass: 'currency', template: '{base}{quote}=X' }          // EUR → EURUSD=X
{ assetClass: 'stock', market: 'TO', template: '{base}.TO' }     // RY → RY.TO
{ template: '{base}' }                                            // AAPL → AAPL (fallback)

// Finnhub
{ assetClass: 'currency', template: 'OANDA:{base}_{quote}' }     // EUR → OANDA:EUR_USD
{ assetClass: 'crypto', template: 'BINANCE:{base}{quote}' }      // BTC → BINANCE:BTCUSD
{ market: 'NYSE', template: '{base}' }                           // AAPL → AAPL
{ market: 'NASDAQ', template: '{base}' }                         // MSFT → MSFT
{ template: '{base}{exchangeSuffix}' }                           // VOD → VOD.L (fallback)

// Alpaca
{ assetClass: 'crypto', template: '{base}/{quote}' }             // BTC → BTC/USD
{ market: 'NYSE', template: '{base}' }                           // AAPL → AAPL
{ market: 'NASDAQ', template: '{base}' }                         // MSFT → MSFT
{ template: '{base}' }                                           // (fallback)
```

The same listing, say Bitcoin quoted in USD, resolves to `BTC-USD` for Yahoo, `BINANCE:BTCUSD` for Finnhub, and `BTC/USD` for Alpaca. The canonical identity stays the same. Only the final rendered symbol changes per provider.

#### Why Inbound Data Doesn't Need Reverse Mapping

The key insight is that the original canonical context is preserved throughout the request and response cycle. When Studio sends a request for `601988.ss` to Yahoo Finance, it already knows this is listing `TG_LSTG_XXXX` with base `601988` on the Shanghai market. The response data, price bars, volume, and timestamps, are associated back to the canonical listing directly. There's no need to parse `601988.ss` and reverse-engineer which listing it belongs to.

This means adding a new data provider requires zero changes to TradingGoose Market. You define a new set of rules and exchange code mappings in your client, and the canonical identity layer stays untouched.

## What TradingGoose Market Is Now

**TradingGoose Market is a self-hostable, open market reference data platform that acts as a single source of truth for listing identity, exchange metadata, market groupings, and trading hours. Clients like TradingGoose Studio resolve canonical records through its API, then apply provider-specific symbol formatting rules at the edge.**

It manages listings, exchanges, markets, cryptocurrencies, currencies, countries, cities, time zones, blockchain networks, and trading hours through both an admin dashboard and a versioned public API.

![The listings table in TradingGoose Market — browsing stocks across US and international markets with icons, asset class, country, quote currency, and market assignments](./listings-table.png)

The tech stack is deliberately simple: Next.js on Bun, PostgreSQL with Drizzle ORM, Better Auth for sessions and HMAC-signed API keys, and shadcn/ui for the admin interface. One deployable unit, one database, minimal operational overhead.

### Beyond Ticker Identity: Trading Hours, Holidays, and Early Closes

Once we had a relational system linking listings to markets, countries, cities, and time zones, we realized it could manage more than just ticker identity. Trading hours are a natural extension. They're tied to the same market entity, and every trading application eventually needs to know when a market is open.

TradingGoose Market stores trading hours as structured data: regular sessions by day of week, holiday closures, and early close dates with their shortened hours. The admin UI provides a visual weekly calendar where you can drag and configure sessions for each market, scoped by country, asset class, or even individual listing when needed.

![Editing market hours in TradingGoose Market — regular sessions shown as a visual weekly calendar, with holidays and early closes managed alongside](./edit-markethour.png)

This is another area where the market-level grouping pays off. Instead of maintaining separate trading hour records for every segment MIC under NYSE, you configure it once for the market and let the API resolve the most specific match, whether that means a listing-specific schedule, an asset-class override, or the market default.

For teams and individuals building trading applications that connect to multiple data sources, TradingGoose Market removes the need to reinvent ticker identity inside every connector. You curate canonical records once in Market, define provider-specific mapping rules in clients like TradingGoose Studio, and keep the rest of the stack speaking the same language.

# Lightweight Stocksera-Inspired Market Buzz Dashboard Plan

## Objective

Build a lightweight, on-demand market buzz dashboard inspired by Stocksera, without deploying the full Django/MySQL monolith.

The target product should answer:

- Which tickers are getting unusual attention?
- Is the attention retail hype, mainstream/news-driven, or structurally relevant?
- Is sentiment diverging from price?
- Is chatter supported by market-structure signals such as short volume, FTD, borrow fee, or Reg SHO?

## Strategic Decision

Do **not** host Stocksera as-is for this product.

Use this fork as a reference library for:

- Feature ideas
- Source mapping
- Existing analytics concepts
- Visual/dashboard inspiration
- Existing Stocksera route/API inventory

Add a new lightweight app under an isolated directory:

```text
edge/
  frontend/
  worker/
  migrations/
  docs/
  wrangler.toml
```

This avoids inheriting the full Django, MySQL, scheduled-task, user-account, email, and admin surface area.

## Recommended Architecture

```text
Browser
  |
  v
Cloudflare Pages
  Static React/Vite dashboard
  |
  | /api/*
  v
Cloudflare Worker
  Single cache-first API function
  |
  |-- Cloudflare KV
  |     hot response cache
  |
  |-- Cloudflare D1
  |     compact historical aggregate snapshots
  |
  |-- Optional R2
  |     raw payload archive only if later needed
  |
  |-- External sources
        Reddit / WSB
        Stocktwits
        price data
        news APIs
        SEC / FINRA / borrow / FTD sources
```

## Initial Route Design

```http
GET /api/health
GET /api/buzz?symbol=TSLA
GET /api/trending
GET /api/history?symbol=TSLA&range=7d
GET /api/feed?symbol=TSLA
GET /api/sources
```

## Main API Response Shape

```json
{
  "symbol": "TSLA",
  "asOf": "2026-05-09T00:00:00.000Z",
  "price": {
    "last": 182.31,
    "changePct1d": 2.14
  },
  "buzz": {
    "totalMentions": 1842,
    "buzzScore": 82,
    "velocity": 1.7,
    "acceleration": 0.4
  },
  "sentiment": {
    "aggregate": 0.31,
    "retail": 0.48,
    "news": -0.06,
    "respected": 0.12,
    "divergence": 0.54
  },
  "sources": [
    {
      "source": "reddit",
      "class": "retail",
      "mentions": 912,
      "sentiment": 0.52,
      "qualityScore": 38
    },
    {
      "source": "stocktwits",
      "class": "retail",
      "mentions": 481,
      "sentiment": 0.45,
      "qualityScore": 34
    },
    {
      "source": "news",
      "class": "mainstream",
      "mentions": 12,
      "sentiment": -0.06,
      "qualityScore": 82
    }
  ],
  "riskFlags": [
    {
      "code": "RETAIL_NEWS_DIVERGENCE",
      "severity": "medium",
      "message": "Retail sentiment materially more bullish than news sentiment."
    }
  ]
}
```

## What to Extract from Existing Stocksera

Keep the concepts, not the deployment model.

### Keep

| Stocksera concept | New use |
|---|---|
| WSB live mentions | Retail hype signal |
| WSB calls/puts mentions | Options-chatter signal |
| Reddit ticker ranking over time | Retail attention trend |
| Stocktwits trending tickers | Retail/social confirmation |
| Twitter/X trending mentions | Optional later social signal |
| News sentiment | Mainstream information signal |
| Short volume | Market-structure context |
| Failure to deliver | Squeeze/settlement stress context |
| Borrowed shares / borrow fee | Short-pressure context |
| Reg SHO | Exceptional market-structure flag |
| Market summary heatmap | Overview page inspiration |
| Earnings calendar | Catalyst context |
| Trading halts | Risk flag |

### Do Not Keep for v1

- Django templates
- Django user accounts
- Django admin
- MySQL runtime dependency
- `config.yaml` as production config
- email setup
- Swagger/DRF API surface
- full scheduled task estate
- full Python monolith deployment

## Proposed Repo Structure

```text
edge/
  README.md
  wrangler.toml

  worker/
    src/
      index.ts
      router.ts

      sources/
        reddit.ts
        stocktwits.ts
        news.ts
        prices.ts
        shortInterest.ts
        ftd.ts
        borrow.ts

      scoring/
        sentiment.ts
        hypeScore.ts
        sourceQuality.ts
        divergence.ts
        riskFlags.ts

      storage/
        kv.ts
        d1.ts
        cache.ts

      types/
        market.ts
        sources.ts
        responses.ts

      utils/
        symbols.ts
        http.ts
        errors.ts
        time.ts

  frontend/
    package.json
    vite.config.ts
    src/
      main.tsx
      App.tsx
      api.ts
      components/
        Dashboard.tsx
        SymbolHeader.tsx
        BuzzSummary.tsx
        PriceBuzzChart.tsx
        SourceBreakdown.tsx
        SignalTable.tsx
        EventFeed.tsx
        RiskFlags.tsx
      styles/
        tokens.ts
        global.css

  migrations/
    0001_init.sql

  docs/
    product-plan.md
    source-map.md
    scoring-model.md
    deployment.md
```

## D1 Schema Draft

```sql
CREATE TABLE IF NOT EXISTS buzz_snapshots (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  symbol TEXT NOT NULL,
  ts INTEGER NOT NULL,
  source TEXT NOT NULL,
  source_class TEXT NOT NULL,
  mentions INTEGER NOT NULL DEFAULT 0,
  sentiment REAL,
  bullish_pct REAL,
  bearish_pct REAL,
  hype_score REAL,
  quality_score REAL,
  velocity REAL,
  acceleration REAL,
  created_at INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_buzz_symbol_ts
ON buzz_snapshots(symbol, ts);

CREATE INDEX IF NOT EXISTS idx_buzz_source_ts
ON buzz_snapshots(source, ts);

CREATE TABLE IF NOT EXISTS feed_items (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  symbol TEXT NOT NULL,
  ts INTEGER NOT NULL,
  source TEXT NOT NULL,
  source_class TEXT NOT NULL,
  title TEXT,
  url TEXT,
  author TEXT,
  sentiment REAL,
  quality_score REAL,
  engagement_score REAL,
  created_at INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_feed_symbol_ts
ON feed_items(symbol, ts);

CREATE TABLE IF NOT EXISTS trending_snapshots (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  ts INTEGER NOT NULL,
  symbol TEXT NOT NULL,
  rank INTEGER,
  buzz_score REAL,
  mentions INTEGER,
  sentiment REAL,
  retail_score REAL,
  mainstream_score REAL,
  divergence REAL,
  created_at INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_trending_ts
ON trending_snapshots(ts);
```

## KV Cache Policy

Suggested keys:

```text
buzz:TSLA:v1              TTL 300s
feed:TSLA:v1              TTL 900s
history:TSLA:7d:v1        TTL 1800s
trending:v1               TTL 300s
source-status:v1          TTL 3600s
```

Suggested behaviour:

```text
0-5 min old:
  return cached immediately

5-30 min old:
  return cached and refresh opportunistically if possible

>30 min old:
  fetch live, update KV, write D1 snapshot
```

## Scoring Model

### Buzz score

```text
buzz_score =
  normalized_mentions * 0.45
  + mention_velocity * 0.30
  + mention_acceleration * 0.15
  + cross_source_confirmation * 0.10
```

### Retail hype score

```text
retail_hype_score =
  reddit_score * 0.45
  + stocktwits_score * 0.35
  + options_chatter_score * 0.10
  + low_quality_amplification_score * 0.10
```

### Mainstream score

```text
mainstream_score =
  news_mentions * 0.35
  + source_quality * 0.35
  + sentiment_confidence * 0.20
  + recency * 0.10
```

### Divergence

```text
divergence = retail_sentiment - mainstream_sentiment
```

Interpretation:

| Divergence | Meaning |
|---:|---|
| > +0.50 | Retail much more bullish than news/informed sources |
| +0.20 to +0.50 | Retail bullish skew |
| -0.20 to +0.20 | Aligned/neutral |
| -0.50 to -0.20 | Retail bearish skew |
| < -0.50 | Retail panic or negative pile-on |

## Risk Flags

Implement deterministic flags before ML:

- `RETAIL_NEWS_DIVERGENCE`
- `BUZZ_WITHOUT_PRICE_CONFIRMATION`
- `BUZZ_WITH_PRICE_BREAKOUT`
- `SHORT_PRESSURE_ELEVATED`
- `FTD_ELEVATED`
- `BORROW_FEE_SPIKE`
- `OPTIONS_CALL_PUT_IMBALANCE`
- `STOCKTWITS_REDDIT_CONFIRMATION`
- `ONE_SOURCE_ONLY`
- `LOW_QUALITY_HYPE`

## Frontend Plan

Use a dense dark dashboard style.

Pages:

1. Market overview
2. Symbol dashboard
3. Source status monitor

Core components:

- `SymbolHeader`
- `BuzzSummary`
- `PriceBuzzChart`
- `SourceBreakdown`
- `SentimentDivergence`
- `RiskFlags`
- `EventFeed`
- `TrendingTable`

Design language:

- near-black background
- compact stat tiles
- thin gridlines
- small labels
- cyan for neutral data
- green for bullish
- red/ruby for bearish
- orange for warnings
- purple for comparative signal lines

## Implementation Milestones

### Milestone 1 - Skeleton

- Add `/edge` app scaffold
- Deploy Cloudflare Worker
- Deploy static dashboard
- Implement `/api/health`
- Implement mocked `/api/buzz`

### Milestone 2 - KV cache

- Add KV bindings
- Implement `getCachedJson`, `setCachedJson`, `withCache`
- Cache all route responses

### Milestone 3 - D1 snapshots

- Add D1 schema migration
- Implement snapshot writes
- Implement `/api/history`

### Milestone 4 - Stocktwits

- Implement `sources/stocktwits.ts`
- Normalize Stocktwits into `SourceSignal`
- Show Stocktwits signal in `/api/buzz`

### Milestone 5 - Reddit / WSB

- Implement `sources/reddit.ts`
- Add mention counts, ticker rank, sentiment, engagement
- Implement `/api/trending`

### Milestone 6 - Price overlay

- Implement `sources/prices.ts`
- Add price and price-change context
- Overlay price with buzz in frontend

### Milestone 7 - News/mainstream signal

- Implement `sources/news.ts`
- Add source-classification whitelist
- Calculate retail vs mainstream divergence

### Milestone 8 - Market-structure context

- Implement short volume, FTD, borrow fee, Reg SHO inputs
- Add market-structure risk flags

### Milestone 9 - Polish/security

- Put dashboard behind Cloudflare Access if private
- Validate ticker input
- Cap history ranges
- Add rate limiting
- Add source status page
- Add deployment docs

## Security Requirements

- No API keys in browser
- Store secrets as Cloudflare Worker secrets
- Validate symbols with `^[A-Z0-9.\-]{1,10}$`
- Max one symbol per request in v1
- Max 30-day history range in v1
- Max 50 feed items per response
- Return stale cache on upstream failure

## Production Config

Move away from `config.yaml` for the new edge app.

Use Worker secrets:

```text
REDDIT_CLIENT_ID
REDDIT_CLIENT_SECRET
REDDIT_USER_AGENT
STOCKTWITS_TOKEN
FMP_KEY
FINNHUB_KEY
POLYGON_KEY
ADANOS_KEY
```

## Non-goals for v1

- Full Stocksera feature parity
- Deploying the Django app
- MySQL dependency
- User accounts
- Real-time WebSockets
- ML model training
- FinBERT pipeline
- Full raw post archive
- Public API product
- Twitter/X as a launch blocker

## Bottom Line

Keep Stocksera as the research and analytics reference. Add a new `/edge` app for the deployable product:

```text
Cloudflare Pages + Worker + KV + D1
```

The first useful version should deliver:

- Stocktwits + Reddit retail hype
- price context
- trending tickers
- cached symbol dashboards
- D1 historical snapshots
- retail vs mainstream divergence once news is added
- deterministic market-risk flags

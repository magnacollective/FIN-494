# Product Requirements Document (PRD)
## Sentiment Cascade Detection Platform
### Detecting Real-Time Emotional Waves Across Stocks and Crypto Markets

**Project Type:** FIN 494 Academic Research Project + Production Dashboard
**Timeline:** 4 Weeks
**Tech Stack:** Railway (Microservices) + Vercel (Frontend) + Reddit API + Multiple Data Sources

---

## 📋 Executive Summary

This project builds a real-time sentiment analysis platform that detects "sentiment cascades" - rapid emotional waves in online investor discussions that precede market volatility. By analyzing social media (Reddit, StockTwits), financial news, and market data, we aim to identify early warning signals of abnormal trading activity across 10 stocks and 5 cryptocurrencies.

**Core Innovation:** Hybrid sentiment analysis (VADER + FinBERT) combined with multi-signal cascade detection algorithm that adapts to each asset's unique volatility profile.

**Deliverables:**
1. Research paper analyzing predictive power of sentiment cascades
2. Production-ready dashboard with near-real-time monitoring (15-30 min updates)
3. Comparative analysis: Do cascades behave differently in stocks vs crypto?

---

## 🎯 Problem Statement & Research Questions

### Primary Research Question
*Can sudden shifts in online investor sentiment serve as early-warning indicators of abnormal market activity, and do they behave differently between stock markets and crypto markets?*

### Secondary Research Questions
1. When do online sentiment cascades occur most frequently?
2. How predictive are sentiment velocity spikes of price volatility (next 1hr, 6hr, 24hr)?
3. Do cascades in crypto markets dissipate faster than in stock markets?
4. Can we detect artificial hype patterns (pump-and-dumps, rug pulls) using posting behavior fingerprints?
5. Does sentiment divergence between retail (Reddit) and institutional (News) sources predict reversals?

### Hypotheses
- **H1:** Sentiment cascades precede volatility spikes by 30-90 minutes on average
- **H2:** Crypto cascades have 2-3× faster velocity than stock cascades
- **H3:** Volume-weighted sentiment is more predictive than raw sentiment
- **H4:** Retail-institutional sentiment divergence >0.4 predicts mean reversion within 24hrs

---

## 👥 User Personas

### 1. Professor/Evaluator (Primary for Week 4)
- **Needs:** Clear methodology, academic rigor, reproducible results
- **Uses:** Dashboard to validate research findings, export data for paper appendix
- **Success:** Understands cascade detection algorithm, sees stocks vs crypto comparison

### 2. Retail Trader (Secondary)
- **Needs:** Actionable alerts, understand why alerts triggered, historical context
- **Uses:** Check dashboard for unusual activity before making trades
- **Success:** Avoided a pump-and-dump or caught early momentum

### 3. Financial Researcher (Tertiary)
- **Needs:** Explore data patterns, test hypotheses, compare assets
- **Uses:** Historical playback, export features, correlation analysis
- **Success:** Published follow-up research using your methodology

---

## 🏗️ Technical Architecture

### **Microservices Architecture on Railway**

```
┌─────────────────────────────────────────────────────────────┐
│                     VERCEL FRONTEND                         │
│  Next.js Dashboard + TailwindCSS + Recharts/Plotly         │
└────────────────┬────────────────────────────────────────────┘
                 │ HTTPS
                 ▼
┌─────────────────────────────────────────────────────────────┐
│              RAILWAY: API GATEWAY SERVICE                   │
│  Node.js/Express - Rate limiting, auth, routing             │
│  Routes: /api/tickers, /api/sentiment, /api/cascades       │
└─┬───────────┬─────────────┬──────────────┬─────────────────┘
  │           │             │              │
  ▼           ▼             ▼              ▼
┌───────┐ ┌────────┐ ┌──────────┐ ┌─────────────────┐
│ DATA  │ │SENTIMENT│ │ CASCADE  │ │ ALERT ENGINE    │
│COLLECT│ │PROCESSOR│ │ DETECTOR │ │ Checks cascade  │
│       │ │         │ │          │ │ scores every    │
│Reddit │ │Hybrid:  │ │Multi-sig │ │ 30min, stores   │
│StockT │ │VADER+   │ │+ Pump    │ │ alerts in DB    │
│News   │ │FinBERT  │ │ patterns │ │                 │
│Finance│ │         │ │          │ │                 │
└───┬───┘ └────┬────┘ └─────┬────┘ └────────┬────────┘
    │          │            │               │
    └──────────┴────────────┴───────────────┘
                       │
                       ▼
    ┌────────────────────────────────────────┐
    │    RAILWAY: PostgreSQL Database        │
    │  Tables: tickers, sentiment_scores,    │
    │  cascades, alerts, raw_posts (temp)    │
    └────────────────────────────────────────┘
                       │
                       ▼
    ┌────────────────────────────────────────┐
    │    RAILWAY: Redis Cache                │
    │  Rate limits, temp aggregations,       │
    │  real-time counters                    │
    └────────────────────────────────────────┘
```

### **Service Breakdown**

#### 1. Data Collector Service (Python/FastAPI)
**Responsibilities:**
- Fetch data from Reddit API (r/wallstreetbets, r/stocks, r/cryptocurrency)
- Fetch from StockTwits API (ticker-specific posts)
- Fetch from NewsAPI (financial headlines)
- Fetch market data (yfinance for stocks, CoinGecko for crypto)

**Cron Schedule:**
- Social data: Every 30 minutes
- Market data: Every 15 minutes (during market hours)
- News: Every 60 minutes

**Reddit Collection Strategy:**
- Keywords per ticker: `{"GME": ["gamestop", "gme"], "TSLA": ["tesla", "tsla", "elon"], ...}`
- Weight by: `score = upvotes * 1.0 + comments * 0.5`
- Minimum threshold: 10 upvotes or 5 comments
- Limit: Top 100 posts per ticker per interval

**Output:** Stores raw posts temporarily in PostgreSQL `raw_posts` table

#### 2. Sentiment Processor Service (Python/FastAPI)
**Responsibilities:**
- Process raw posts through hybrid sentiment pipeline
- Aggregate sentiment scores by ticker + timewindow
- Calculate engagement-weighted averages

**Hybrid Pipeline:**
```python
def analyze_sentiment(text, source):
    # Quick VADER for all text
    vader_score = vader.polarity_scores(text)['compound']

    # FinBERT for news or complex financial language
    if source == 'news' or contains_financial_terms(text):
        finbert_score = finbert_pipeline(text)[0]['score']
        return {
            'vader': vader_score,
            'finbert': finbert_score,
            'final': finbert_score  # Trust FinBERT more
        }

    return {
        'vader': vader_score,
        'finbert': None,
        'final': vader_score
    }

def aggregate_ticker_sentiment(ticker, timewindow):
    posts = get_posts(ticker, timewindow)
    weighted_sum = sum(post.sentiment * post.engagement_score for post in posts)
    total_weight = sum(post.engagement_score for post in posts)

    return {
        'ticker': ticker,
        'timestamp': timewindow.end,
        'sentiment': weighted_sum / total_weight,
        'post_count': len(posts),
        'total_engagement': total_weight,
        'source_breakdown': {
            'reddit': reddit_sentiment,
            'stocktwits': stocktwits_sentiment,
            'news': news_sentiment
        }
    }
```

**Output:** Stores in `sentiment_scores` table

#### 3. Cascade Detector Service (Node.js/TypeScript)
**Responsibilities:**
- Calculate sentiment velocity, volume deltas, engagement deltas
- Compute multi-signal cascade scores
- Detect pump-and-dump patterns
- Update adaptive baselines daily

**Multi-Signal Algorithm:**
```javascript
function detectCascade(ticker, currentData, historicalBaseline) {
  // Calculate z-scores (standard deviations from mean)
  const sentimentDelta = Math.abs(
    (currentData.sentiment - historicalBaseline.sentiment_mean) /
    historicalBaseline.sentiment_std
  );

  const volumeDelta = (
    (currentData.post_count - historicalBaseline.volume_mean) /
    historicalBaseline.volume_std
  );

  const engagementDelta = (
    (currentData.avg_engagement - historicalBaseline.engagement_mean) /
    historicalBaseline.engagement_std
  );

  // Weighted cascade score
  const cascadeScore = (
    0.40 * sentimentDelta +
    0.35 * volumeDelta +
    0.25 * engagementDelta
  );

  // Adaptive threshold based on recent volatility
  const baseThreshold = 5.0;
  const volatilityMultiplier = getVolatilityMultiplier(ticker);
  const threshold = baseThreshold * volatilityMultiplier;

  // Alert levels
  let alertLevel = null;
  if (cascadeScore > threshold * 1.6) alertLevel = 'RED';
  else if (cascadeScore > threshold * 1.2) alertLevel = 'ORANGE';
  else if (cascadeScore > threshold) alertLevel = 'YELLOW';

  // Check for pump patterns
  const pumpPattern = detectPumpPattern(ticker, currentData);
  if (pumpPattern.detected) alertLevel = 'RED';

  return {
    ticker,
    timestamp: currentData.timestamp,
    cascadeScore,
    alertLevel,
    components: { sentimentDelta, volumeDelta, engagementDelta },
    pumpPattern
  };
}

function detectPumpPattern(ticker, recentData) {
  // Pattern: Rapid positive spike + concentrated posting + repetitive text
  const last30min = recentData.slice(-1);
  const last2hrs = recentData.slice(-4);

  const rapidSpike = last30min[0].sentiment > 0.6 &&
                     (last30min[0].sentiment - last2hrs[0].sentiment) > 0.5;

  const concentratedPosting = (
    last30min[0].unique_users / last30min[0].post_count < 0.3
  );

  const repetitivePhrasing = calculateCosineSimilarity(
    last30min[0].top_phrases
  ) > 0.7;

  return {
    detected: rapidSpike && concentratedPosting && repetitivePhrasing,
    confidence: (rapidSpike ? 0.4 : 0) +
                (concentratedPosting ? 0.3 : 0) +
                (repetitivePhrasing ? 0.3 : 0)
  };
}
```

**Baseline Calculation:**
```javascript
function calculateAdaptiveBaseline(ticker) {
  const data_30day = getHistoricalData(ticker, 30);
  const data_7day = getHistoricalData(ticker, 7);
  const data_90day = getHistoricalData(ticker, 90);

  const baseline_30day = {
    sentiment_mean: mean(data_30day.sentiment),
    sentiment_std: std(data_30day.sentiment),
    volume_mean: mean(data_30day.post_count),
    volume_std: std(data_30day.post_count),
    engagement_mean: mean(data_30day.engagement),
    engagement_std: std(data_30day.engagement)
  };

  // Adaptive boost based on recent volatility
  const recent_volatility = std(data_7day.sentiment);
  const historical_volatility = std(data_90day.sentiment);

  let volatilityMultiplier = 1.0;
  if (recent_volatility > 2 * historical_volatility) {
    volatilityMultiplier = 1.25; // Market already excited
  } else if (recent_volatility < 0.5 * historical_volatility) {
    volatilityMultiplier = 0.85; // Market quiet, smaller moves matter
  }

  return { baseline_30day, volatilityMultiplier };
}
```

**Output:** Stores in `cascades` table

#### 4. Alert Engine Service (Node.js)
**Responsibilities:**
- Query cascade detector results every 30 min
- Generate alerts for dashboard
- Track alert history for research analysis

**Output:** Stores in `alerts` table

#### 5. API Gateway Service (Node.js/Express)
**Responsibilities:**
- Serve REST API to Vercel frontend
- Handle authentication (if needed)
- Rate limiting (Redis-backed)
- CORS configuration

**Endpoints:**
```
GET  /api/tickers                    # List all tracked tickers
GET  /api/ticker/:symbol             # Get ticker details + current sentiment
GET  /api/sentiment/:symbol          # Time-series sentiment data
GET  /api/cascades                   # Current active cascades
GET  /api/cascades/history           # Historical cascades (for research)
GET  /api/alerts                     # Current alerts
GET  /api/comparison                 # Stocks vs crypto aggregate stats
GET  /api/divergence/:symbol         # Retail vs institutional sentiment
POST /api/export                     # Export data for research paper
```

---

## 📊 Database Schema

### PostgreSQL Tables

```sql
-- Tracked tickers
CREATE TABLE tickers (
  id SERIAL PRIMARY KEY,
  symbol VARCHAR(10) UNIQUE NOT NULL,
  name VARCHAR(100),
  asset_type VARCHAR(10) CHECK (asset_type IN ('stock', 'crypto')),
  market_cap BIGINT,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Aggregated sentiment scores (main analytics table)
CREATE TABLE sentiment_scores (
  id SERIAL PRIMARY KEY,
  ticker_id INTEGER REFERENCES tickers(id),
  timestamp TIMESTAMP NOT NULL,

  -- Sentiment metrics
  sentiment_score DECIMAL(5,4),      -- -1 to 1
  vader_score DECIMAL(5,4),
  finbert_score DECIMAL(5,4),

  -- Volume metrics
  post_count INTEGER,
  unique_users INTEGER,
  total_engagement DECIMAL(10,2),    -- Weighted upvotes + comments

  -- Source breakdown
  reddit_sentiment DECIMAL(5,4),
  stocktwits_sentiment DECIMAL(5,4),
  news_sentiment DECIMAL(5,4),
  reddit_volume INTEGER,
  stocktwits_volume INTEGER,
  news_volume INTEGER,

  -- Market data (for correlation analysis)
  price DECIMAL(15,6),
  volume BIGINT,
  price_change_1hr DECIMAL(8,4),

  created_at TIMESTAMP DEFAULT NOW(),
  UNIQUE(ticker_id, timestamp)
);

CREATE INDEX idx_sentiment_ticker_time ON sentiment_scores(ticker_id, timestamp DESC);

-- Cascade events
CREATE TABLE cascades (
  id SERIAL PRIMARY KEY,
  ticker_id INTEGER REFERENCES tickers(id),
  detected_at TIMESTAMP NOT NULL,

  -- Cascade metrics
  cascade_score DECIMAL(8,4),
  alert_level VARCHAR(10) CHECK (alert_level IN ('YELLOW', 'ORANGE', 'RED')),

  -- Component scores
  sentiment_delta DECIMAL(8,4),
  volume_delta DECIMAL(8,4),
  engagement_delta DECIMAL(8,4),

  -- Pattern detection
  pump_pattern_detected BOOLEAN DEFAULT FALSE,
  pump_confidence DECIMAL(4,3),

  -- Outcome tracking (for research)
  price_change_1hr DECIMAL(8,4),
  price_change_6hr DECIMAL(8,4),
  price_change_24hr DECIMAL(8,4),
  volume_change_1hr DECIMAL(8,4),

  -- Resolution
  resolved_at TIMESTAMP,
  resolution_type VARCHAR(20),  -- 'volatility_spike', 'false_alarm', 'pump_dump', etc.

  created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_cascades_ticker_detected ON cascades(ticker_id, detected_at DESC);

-- Alerts (for dashboard)
CREATE TABLE alerts (
  id SERIAL PRIMARY KEY,
  cascade_id INTEGER REFERENCES cascades(id),
  ticker_id INTEGER REFERENCES tickers(id),
  alert_level VARCHAR(10),
  message TEXT,
  is_read BOOLEAN DEFAULT FALSE,
  created_at TIMESTAMP DEFAULT NOW()
);

-- Baselines (adaptive thresholds)
CREATE TABLE baselines (
  id SERIAL PRIMARY KEY,
  ticker_id INTEGER REFERENCES tickers(id),
  calculated_at TIMESTAMP NOT NULL,

  -- 30-day baseline
  sentiment_mean_30d DECIMAL(5,4),
  sentiment_std_30d DECIMAL(5,4),
  volume_mean_30d DECIMAL(10,2),
  volume_std_30d DECIMAL(10,2),
  engagement_mean_30d DECIMAL(10,2),
  engagement_std_30d DECIMAL(10,2),

  -- Volatility multiplier
  volatility_multiplier DECIMAL(4,3),

  UNIQUE(ticker_id, calculated_at)
);

-- Raw posts (temporary storage, cleared after 7 days)
CREATE TABLE raw_posts (
  id SERIAL PRIMARY KEY,
  ticker_id INTEGER REFERENCES tickers(id),
  source VARCHAR(20),  -- 'reddit', 'stocktwits', 'news'
  external_id VARCHAR(100),
  text TEXT,
  author VARCHAR(100),
  created_at TIMESTAMP,
  upvotes INTEGER DEFAULT 0,
  comments INTEGER DEFAULT 0,
  engagement_score DECIMAL(10,2),
  collected_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_raw_posts_ticker_created ON raw_posts(ticker_id, created_at DESC);

-- Auto-delete old raw posts (keep only 7 days)
CREATE OR REPLACE FUNCTION delete_old_posts()
RETURNS void AS $$
BEGIN
  DELETE FROM raw_posts WHERE collected_at < NOW() - INTERVAL '7 days';
END;
$$ LANGUAGE plpgsql;
```

---

## 🎨 Feature Requirements (MoSCoW Method)

### **MUST HAVE (Week 1-3)**
- ✅ Data collection from Reddit, StockTwits, NewsAPI every 30 min
- ✅ Hybrid sentiment analysis (VADER + FinBERT)
- ✅ Multi-signal cascade detection with adaptive thresholds
- ✅ Track 10 stocks (GME, AMC, BBBY, TSLA, NVDA, PLTR, AAPL, MSFT, SPY, SOL) + 5 crypto (BTC, ETH, DOGE, SHIB, SOL)
- ✅ Dashboard features:
  - Real-time sentiment vs price charts (Feature A)
  - Cascade detection alerts (Feature B)
  - Stock vs crypto comparison view (Feature C)
  - Export data functionality (Feature G)
- ✅ PostgreSQL database with proper schema
- ✅ REST API for frontend consumption
- ✅ Basic authentication/rate limiting

### **SHOULD HAVE (Week 3-4)**
- 🎯 Pump-and-dump pattern detection
- 🎯 Retail vs institutional sentiment divergence tracking
- 🎯 Historical playback feature (select date range, replay cascades)
- 🎯 Alert summary dashboard (active alerts, alert history)
- 🎯 Correlation analysis (sentiment velocity → price change)
- 🎯 Mobile-responsive dashboard

### **COULD HAVE (If time permits)**
- 💡 Email notifications for RED alerts
- 💡 Custom ticker watchlist (user-configurable)
- 💡 Sentiment breakdown by subreddit
- 💡 Top phrases/keywords per cascade
- 💡 Social network analysis (who's driving the cascade?)
- 💡 Integration with TradingView charts

### **WON'T HAVE (Out of scope)**
- ❌ Live trading integration
- ❌ Mobile app (web only)
- ❌ User accounts/authentication (unless required)
- ❌ Paid API integrations
- ❌ Real-time websocket streaming (30min batch is enough)

---

## 🔄 Data Pipeline Flow

```
┌─────────────────────────────────────────────────────────┐
│  EVERY 30 MINUTES (Cron Job)                            │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 1: DATA COLLECTION (5-10 min)                     │
│  - Reddit API: /r/wallstreetbets, /r/stocks hot + new   │
│  - StockTwits: Search by ticker symbols                 │
│  - NewsAPI: Financial headlines for each ticker         │
│  - Store in raw_posts table                             │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 2: SENTIMENT PROCESSING (3-5 min)                 │
│  - VADER on all posts (fast)                            │
│  - FinBERT on news + complex posts                      │
│  - Weight by engagement (upvotes, comments)             │
│  - Aggregate by ticker + 30min window                   │
│  - Store in sentiment_scores table                      │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 3: MARKET DATA FETCH (1-2 min)                    │
│  - yfinance: Current stock prices, volume               │
│  - CoinGecko: Crypto prices, 24hr change                │
│  - Update sentiment_scores with price data              │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 4: CASCADE DETECTION (1-2 min)                    │
│  - Calculate deltas vs baseline                         │
│  - Compute cascade scores                               │
│  - Detect pump patterns                                 │
│  - Generate alerts if threshold exceeded                │
│  - Store in cascades + alerts tables                    │
└─────────────────────────────────────────────────────────┘
                        │
                        ▼
┌─────────────────────────────────────────────────────────┐
│  STEP 5: OUTCOME TRACKING (async, runs 1hr/6hr/24hr later)│
│  - Update cascade records with price changes            │
│  - Calculate correlation metrics for research           │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│  DAILY AT MIDNIGHT                                       │
│  - Recalculate adaptive baselines (30/7/90 day windows) │
│  - Delete raw_posts older than 7 days                   │
│  - Generate daily summary stats                         │
└─────────────────────────────────────────────────────────┘
```

---

## 📱 Dashboard UI/UX Requirements

### **Landing Page**
```
┌────────────────────────────────────────────────────────────┐
│  SENTIMENT CASCADE DETECTOR                    [Export]   │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  🔴 ACTIVE ALERTS (3)                                      │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ 🔴 GME - RED ALERT - Pump Pattern Detected           │ │
│  │ Score: 8.4 | +0.62 sentiment | 5.2× volume           │ │
│  │ Detected: 2 min ago                        [Details] │ │
│  └──────────────────────────────────────────────────────┘ │
│  ┌──────────────────────────────────────────────────────┐ │
│  │ 🟠 DOGE - ORANGE ALERT - Sentiment Cascade           │ │
│  │ Score: 6.8 | +0.41 sentiment | 3.1× volume           │ │
│  │ Detected: 28 min ago                       [Details] │ │
│  └──────────────────────────────────────────────────────┘ │
│                                                            │
│  📊 MONITORED ASSETS                                       │
│  [Stocks] [Crypto] [All]                                  │
│                                                            │
│  ┌──────┬──────────┬───────────┬─────────┬──────────────┐ │
│  │Ticker│ Sentiment│ Velocity  │ Volume  │ Cascade Score│ │
│  ├──────┼──────────┼───────────┼─────────┼──────────────┤ │
│  │ GME  │ +0.68 🔺 │ +0.52/hr  │ 847 📈  │ 8.4 🔴      │ │
│  │ TSLA │ +0.23 —  │ +0.04/hr  │ 234 —   │ 2.1 ⚪      │ │
│  │ BTC  │ -0.11 🔻 │ -0.08/hr  │ 412 —   │ 1.8 ⚪      │ │
│  │ ...  │ ...      │ ...       │ ...     │ ...          │ │
│  └──────┴──────────┴───────────┴─────────┴──────────────┘ │
│                                            [View All →]    │
└────────────────────────────────────────────────────────────┘
```

### **Ticker Detail Page** (Feature A: Sentiment vs Price)
```
┌────────────────────────────────────────────────────────────┐
│  ← Back to Dashboard              GME - GameStop Corp      │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Current Status: 🔴 RED ALERT - Pump Pattern Detected     │
│  Cascade Score: 8.4 | Price: $24.56 (+3.2%)               │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │        SENTIMENT vs PRICE (Last 24 Hours)          │   │
│  │                                                    │   │
│  │  Sentiment                              Price      │   │
│  │   1.0 ┤                                    $26     │   │
│  │   0.8 ┤         ╱╲                         $25     │   │
│  │   0.6 ┤    🔴──╯  ╲                        $24     │   │
│  │   0.4 ┤   ╱        ╲___                    $23     │   │
│  │   0.2 ┤──╯                ╲                $22     │   │
│  │   0.0 ┼────────────────────────────────    $21     │   │
│  │       12am   6am   12pm   6pm   12am                │   │
│  │                                                    │   │
│  │  ━━ Sentiment    ━━ Price                         │   │
│  └────────────────────────────────────────────────────┘   │
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │          CASCADE DETECTION TIMELINE                │   │
│  │                                                    │   │
│  │  🟡 Yellow Alert - 6:30am - Score 5.2             │   │
│  │  🟠 Orange Alert - 8:45am - Score 6.8             │   │
│  │  🔴 RED Alert - 10:12am - Score 8.4 (CURRENT)     │   │
│  │     ↳ Pump Pattern: 87% confidence                │   │
│  │     ↳ Concentrated posting: 67% from 5 users      │   │
│  │     ↳ Repetitive phrases detected                 │   │
│  └────────────────────────────────────────────────────┘   │
│                                                            │
│  SENTIMENT BREAKDOWN BY SOURCE                             │
│  Reddit: +0.74 (532 posts) | StockTwits: +0.61 (89 posts) │
│  News: +0.12 (3 articles)                                  │
│                                                            │
│  RETAIL vs INSTITUTIONAL DIVERGENCE                        │
│  Retail: +0.68 | Institutional: +0.12 | Divergence: 0.56  │
│  ⚠️ High divergence - potential reversal signal            │
│                                                            │
│  [Export Data] [View Historical Cascades]                  │
└────────────────────────────────────────────────────────────┘
```

### **Comparison View** (Feature C: Stocks vs Crypto)
```
┌────────────────────────────────────────────────────────────┐
│  STOCKS vs CRYPTO COMPARISON                               │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Time Range: [Last 7 Days ▼]                              │
│                                                            │
│  ┌───────────────────────┬───────────────────────┐        │
│  │      STOCKS           │       CRYPTO          │        │
│  ├───────────────────────┼───────────────────────┤        │
│  │ Avg Cascade Score     │ Avg Cascade Score     │        │
│  │      3.2              │      5.7              │        │
│  │                       │                       │        │
│  │ Cascade Frequency     │ Cascade Frequency     │        │
│  │   12 events/week      │   28 events/week      │        │
│  │                       │                       │        │
│  │ Avg Velocity          │ Avg Velocity          │        │
│  │  +0.08 per hour       │  +0.21 per hour       │        │
│  │                       │                       │        │
│  │ Sentiment→Price R²    │ Sentiment→Price R²    │        │
│  │      0.42             │      0.68             │        │
│  └───────────────────────┴───────────────────────┘        │
│                                                            │
│  KEY INSIGHTS:                                             │
│  • Crypto cascades are 2.3× more frequent than stocks     │
│  • Crypto sentiment velocity is 2.6× faster               │
│  • Crypto shows stronger sentiment-price correlation      │
│  • Stock cascades have longer duration (avg 4.2hrs vs 1.8hrs)│
│                                                            │
│  ┌────────────────────────────────────────────────────┐   │
│  │     CASCADE DISTRIBUTION (Last 30 Days)            │   │
│  │                                                    │   │
│  │  Count                                             │   │
│  │   30 ┤                                              │   │
│  │   25 ┤              ██                              │   │
│  │   20 ┤              ██                              │   │
│  │   15 ┤       ██     ██     ██                      │   │
│  │   10 ┤ ██    ██     ██     ██    ██                │   │
│  │    5 ┤ ██    ██     ██     ██    ██    ██          │   │
│  │    0 ┼──────────────────────────────────────       │   │
│  │       GME  TSLA   BTC   DOGE  ETH   SHIB           │   │
│  │                                                    │   │
│  │  ██ Stocks    ██ Crypto                            │   │
│  └────────────────────────────────────────────────────┘   │
└────────────────────────────────────────────────────────────┘
```

### **Export Feature** (Feature G)
```
┌────────────────────────────────────────────────────────────┐
│  EXPORT DATA FOR RESEARCH                                  │
├────────────────────────────────────────────────────────────┤
│                                                            │
│  Select Data Range:                                        │
│  From: [2024-01-01]  To: [2024-04-15]                     │
│                                                            │
│  Select Tickers: [☑ All] or:                              │
│  ☑ GME  ☑ AMC  ☑ TSLA  ☑ BTC  ☑ ETH  ...                │
│                                                            │
│  Select Data Types:                                        │
│  ☑ Sentiment scores (aggregated)                          │
│  ☑ Cascade events                                         │
│  ☑ Alert history                                          │
│  ☑ Price correlation data                                 │
│  ☐ Raw posts (warning: large file)                        │
│                                                            │
│  Export Format:                                            │
│  ◉ CSV  ○ JSON  ○ Excel (.xlsx)                           │
│                                                            │
│  Include Analysis:                                         │
│  ☑ Summary statistics                                     │
│  ☑ Correlation matrices                                   │
│  ☑ Cascade outcome tracking                               │
│                                                            │
│                           [Generate Export]                │
│                                                            │
│  Estimated file size: 4.2 MB                               │
│  Processing time: ~30 seconds                              │
└────────────────────────────────────────────────────────────┘
```

---

## 📈 Success Metrics & KPIs

### **Technical Metrics**
- **Data Collection Success Rate:** >95% successful API calls
- **Sentiment Processing Speed:** <5 min per 30-min batch
- **Dashboard Load Time:** <2 seconds
- **API Uptime:** >99% (Railway reliability)
- **Database Query Performance:** <500ms for most queries

### **Research Metrics**
- **Cascade Detection Accuracy:**
  - Precision: % of alerts that led to volatility (target: >60%)
  - Recall: % of volatility events preceded by alerts (target: >50%)
- **False Positive Rate:** <40% of alerts
- **Predictive Power:**
  - Sentiment → Price Correlation (target R² > 0.4 for crypto, >0.3 for stocks)
  - Lead time: Cascades precede volatility by 30-120 min (hypothesis)
- **Stocks vs Crypto Comparison:**
  - Measurable difference in cascade velocity (target: >1.5× faster in crypto)
  - Different optimal thresholds per asset class

### **Academic Deliverable Metrics**
- **Research Paper:** 15-20 pages, publishable quality
- **Methodology Reproducibility:** Full documentation, open-source code
- **Data Visualization Quality:** Publication-ready charts
- **Novelty:** Unique contribution (adaptive thresholds, hybrid sentiment, cross-market comparison)

### **Dashboard User Metrics** (if time permits)
- **Alert Usefulness:** User feedback on alert quality
- **Feature Usage:** Which features get used most (A/B/C/G)
- **Export Downloads:** How many times data exported for analysis

---

## 📅 Timeline & Milestones (4 Weeks)

### **Week 1: Foundation & Data Collection** (Nov 4 - Nov 10)
**Deliverables:**
- ✅ Railway project setup (all 5 microservices)
- ✅ PostgreSQL schema deployed
- ✅ Data collector service running (Reddit, StockTwits, News)
- ✅ Initial dataset: 7 days of posts for 15 tickers
- ✅ Basic API endpoints functional

**Tasks:**
- Day 1-2: Railway infrastructure setup, database design
- Day 3-4: Implement data collectors (Reddit API integration critical)
- Day 5-6: Run collectors, validate data quality
- Day 7: API gateway setup, test endpoints

**Milestone:** "Data Pipeline Operational" - 10k+ posts collected

---

### **Week 2: Sentiment & Cascade Detection** (Nov 11 - Nov 17)
**Deliverables:**
- ✅ Hybrid sentiment analysis working (VADER + FinBERT)
- ✅ Sentiment aggregation by ticker + time window
- ✅ Cascade detection algorithm implemented
- ✅ Adaptive baseline calculation
- ✅ First cascade alerts generated

**Tasks:**
- Day 1-2: VADER integration, test on collected posts
- Day 3-4: FinBERT setup (Hugging Face or self-hosted), hybrid pipeline
- Day 5: Cascade detector implementation
- Day 6: Baseline calculation, tune thresholds
- Day 7: Generate first week of cascade events for analysis

**Milestone:** "Cascade Detection Live" - 20+ cascades detected

---

### **Week 3: Dashboard & Analysis** (Nov 18 - Nov 24)
**Deliverables:**
- ✅ Vercel frontend deployed
- ✅ Features A, B, C implemented (sentiment charts, alerts, comparison)
- ✅ Historical playback working
- ✅ Pump-and-dump pattern detection
- ✅ Initial research analysis (correlation, frequency stats)

**Tasks:**
- Day 1-2: Next.js dashboard setup, connect to API
- Day 3-4: Implement ticker detail page (charts, timelines)
- Day 5: Comparison view (stocks vs crypto)
- Day 6: Alert system UI, pump pattern detection
- Day 7: Run analysis on 2-3 weeks of data

**Milestone:** "Dashboard Demo-Ready" - All must-have features working

---

### **Week 4: Polish, Research Paper, Presentation** (Nov 25 - Dec 1)
**Deliverables:**
- ✅ Feature G: Export functionality
- ✅ Research paper drafted (15-20 pages)
- ✅ Data visualizations for paper
- ✅ Presentation slides
- ✅ Video demo (if required)
- ✅ Final testing & bug fixes

**Tasks:**
- Day 1-2: Export feature, final dashboard polish
- Day 3-4: Write research paper (methodology, results, discussion)
- Day 5: Create visualizations, charts for paper
- Day 6: Presentation prep, demo video recording
- Day 7: Final review, submission

**Milestone:** "Project Complete" - Ready for FIN 494 submission

---

## 💰 Budget Estimates

### **Railway Costs** (Microservices)
- **PostgreSQL Database:** $5-10/month (1GB storage, sufficient)
- **Redis Cache:** $3-5/month (256MB)
- **Data Collector Service:** $5-7/month (runs every 30 min)
- **Sentiment Processor:** $8-12/month (CPU-intensive for FinBERT)
- **Cascade Detector:** $3-5/month (lightweight compute)
- **Alert Engine:** $2-3/month
- **API Gateway:** $3-5/month

**Total Railway:** ~$30-45/month
**4-week project:** ~$35-50 total
**Student discount:** Railway offers $5 free/month → ~$25-40 actual cost

### **API Costs** (All Free Tiers)
- **Reddit API:** FREE (60 requests/min = 1800/30min window - plenty)
- **StockTwits API:** FREE (200 requests/hour)
- **NewsAPI:** FREE (100 requests/day)
- **yfinance (Yahoo Finance):** FREE (unofficial but reliable)
- **CoinGecko API:** FREE (50 calls/min)
- **Hugging Face Inference API:** FREE (<30k chars/month likely sufficient)

**Total API Costs:** $0

### **Vercel (Frontend)**
- **Hosting:** FREE (Hobby tier, unlimited bandwidth)
- **Serverless Functions:** FREE (100GB-hours/month)

**Total Vercel:** $0

### **Domain (Optional)**
- sentiment-cascade.com: ~$12/year (optional, can use vercel.app subdomain)

---

## 🎓 Total Project Cost: $25-50 for 4 weeks

**Cost Optimization Tips:**
1. Use Railway free $5/month credit
2. Pause non-essential services during downtime
3. Reduce data collection frequency to every 60 min (half the compute)
4. Use Vercel's free tier (more than enough for academic project)

---

## ⚠️ Risks & Mitigations

### **Risk 1: Reddit API Rate Limits**
**Impact:** HIGH - Reddit is primary data source
**Probability:** MEDIUM
**Mitigation:**
- Implement exponential backoff
- Respect 60 req/min limit (1 req/sec)
- Cache results in Redis
- Use PRAW library (handles rate limits)
- Fallback: Reduce ticker count if limits hit

### **Risk 2: FinBERT Processing Too Slow**
**Impact:** MEDIUM - Could delay pipeline
**Probability:** MEDIUM
**Mitigation:**
- Batch processing (50 posts at once)
- Use Hugging Face API instead of self-hosting
- Fallback: Use VADER only (still academically valid)
- Optimize: Only FinBERT on news, VADER on social

### **Risk 3: Railway Costs Exceed Budget**
**Impact:** MEDIUM - Project cost overrun
**Probability:** LOW
**Mitigation:**
- Monitor usage daily via Railway dashboard
- Set billing alerts at $30
- Reduce data collection frequency if needed
- Combine microservices if costs too high

### **Risk 4: Cascade Detection False Positives**
**Impact:** MEDIUM - Undermines research validity
**Probability:** HIGH (expected in early testing)
**Mitigation:**
- Tune thresholds using Week 1-2 data
- A/B test different threshold values
- Document false positive rate honestly in paper
- Academic value: Analyzing failure cases is still research

### **Risk 5: Not Enough Time for All Features**
**Impact:** LOW - Can still deliver core value
**Probability:** MEDIUM
**Mitigation:**
- Focus on MUST-HAVE features first
- MoSCoW prioritization already done
- Paper + basic dashboard > fancy dashboard alone
- Pump detection is SHOULD-HAVE, not MUST-HAVE

### **Risk 6: Data Quality Issues**
**Impact:** HIGH - Garbage in, garbage out
**Probability:** MEDIUM
**Mitigation:**
- Data validation on collection (check for nulls, outliers)
- Manual spot-checks daily
- Filter spam/bot posts (min engagement threshold)
- Store raw data for re-processing if needed

### **Risk 7: Vercel-Railway CORS Issues**
**Impact:** LOW - Technical blocker
**Probability:** LOW
**Mitigation:**
- Configure CORS properly in API Gateway from day 1
- Test locally before deploying
- Use Railway's public networking (not private)

---

## 🚀 Future Enhancements (Post-FIN 494)

If you want to continue this project beyond the class:

1. **Real-Time WebSockets:** Live updates instead of 30-min batches
2. **Machine Learning Cascade Predictor:** Train model on historical cascade → outcome data
3. **Multi-Asset Correlation:** "When GME spikes, AMC follows 73% of the time"
4. **Sentiment Source Attribution:** Which Reddit users are most influential?
5. **Options Flow Integration:** Add unusual options activity data
6. **Mobile App:** React Native dashboard
7. **Public API:** Let other researchers use your data
8. **Academic Publication:** Submit to finance/data science journal
9. **FinTech Startup:** Monetize as SaaS product ($29/mo for retail traders)

---

## 📚 Appendix

### **Tracked Tickers (Final List)**

**Stocks (10):**
1. GME - GameStop Corp (Meme stock)
2. AMC - AMC Entertainment (Meme stock)
3. BBBY - Bed Bath & Beyond (Recent volatility)
4. TSLA - Tesla Inc (High-beta tech)
5. NVDA - NVIDIA Corp (AI hype)
6. PLTR - Palantir Technologies (Reddit favorite)
7. AAPL - Apple Inc (Stable large-cap)
8. MSFT - Microsoft Corp (Stable large-cap)
9. SPY - SPDR S&P 500 ETF (Market proxy)
10. COIN - Coinbase Global (Crypto-related stock)

**Crypto (5):**
1. BTC - Bitcoin (Baseline)
2. ETH - Ethereum (Second largest)
3. DOGE - Dogecoin (Pure meme)
4. SHIB - Shiba Inu (High volatility meme)
5. SOL - Solana (High-performance L1)

### **Reddit Subreddits to Monitor**
- r/wallstreetbets (stocks + some crypto)
- r/stocks (general stock discussion)
- r/cryptocurrency (crypto-specific)
- r/CryptoMarkets (crypto trading)
- r/investing (more conservative sentiment)

### **NewsAPI Keywords**
```javascript
const newsKeywords = {
  stocks: ["stock market", "wall street", "trading", "shares"],
  crypto: ["cryptocurrency", "bitcoin", "crypto market", "blockchain"],
  tickers: ["GME", "GameStop", "Tesla", "Bitcoin", "Ethereum", ...]
};
```

### **Technology Stack Summary**

**Backend (Railway):**
- Python 3.11 (Data collection, FinBERT)
- Node.js 18 (API Gateway, Cascade Detector)
- PostgreSQL 15
- Redis 7
- Libraries: PRAW, Transformers, FastAPI, Express, Prisma/TypeORM

**Frontend (Vercel):**
- Next.js 14 (App Router)
- TypeScript
- TailwindCSS
- Recharts or Plotly (charts)
- SWR or TanStack Query (data fetching)

**APIs:**
- Reddit API (PRAW)
- StockTwits API
- NewsAPI
- yfinance (Yahoo Finance)
- CoinGecko API
- Hugging Face Inference API (optional)

**DevOps:**
- GitHub (version control)
- Railway (microservices deployment)
- Vercel (frontend deployment)
- GitHub Actions (CI/CD, optional)

---

## ✅ Definition of Done

**Project is complete when:**
1. ✅ All 15 tickers collecting data every 30 min for 3+ weeks
2. ✅ Sentiment analysis running on all posts
3. ✅ 50+ cascade events detected and tracked
4. ✅ Dashboard deployed with features A, B, C, G functional
5. ✅ Research paper completed (15-20 pages)
6. ✅ Presentation slides ready
7. ✅ Exportable dataset for professor review
8. ✅ GitHub repo documented (README with setup instructions)
9. ✅ Video demo recorded (5-10 min)
10. ✅ All deliverables submitted to FIN 494

---

## 📞 Support & Resources

**Railway Documentation:** https://docs.railway.app
**Vercel Documentation:** https://vercel.com/docs
**Reddit API (PRAW):** https://praw.readthedocs.io
**FinBERT Model:** https://huggingface.co/ProsusAI/finbert
**VADER Sentiment:** https://github.com/cjhutto/vaderSentiment

**Academic References:**
- Bollen et al. (2011) - Twitter mood predicts stock market
- Sprenger et al. (2014) - Tweets and trades
- Renault (2017) - Intraday online investor sentiment

---

**Document Version:** 1.0
**Last Updated:** November 4, 2024
**Author:** Cruz Flores
**Course:** FIN 494 - Financial Technology & Innovation
**Project Status:** Ready to Build 🚀

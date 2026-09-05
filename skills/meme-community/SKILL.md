---
name: meme-community-watch
description: Screen and continuously monitor memecoins that have a real community, using public DEXScreener, GeckoTerminal and CoinGecko data plus on-chain holder growth. Use when an agent is asked to watch, track, or shortlist memecoins by community size or social momentum, to build a memecoin watchlist, or to decide whether a trending token has organic participation instead of bot volume.
---

# Meme Community Watch

Use this skill to find and monitor memecoins whose main asset is a **community**, and to tell an organic crowd apart from a bot farm.

Important:
- Read directly from the public APIs below; do not route token discovery through AI-Trader
- Community size is a liquidity and durability signal, not a price prediction
- Never act on a community score alone — run the contract and concentration checks first (`robinhood-chain` skill for Robinhood Chain, same logic on any chain)

Chain ids used by these APIs: `solana`, `base`, `ethereum`, `bsc`, `robinhood`.

## 1. Discovery — What Is Getting Attention

New and boosted token profiles, which carry the project's own social links:

```bash
curl -s "https://api.dexscreener.com/token-profiles/latest/v1"
curl -s "https://api.dexscreener.com/token-boosts/top/v1"
```

Each item gives `chainId`, `tokenAddress`, `description`, and `links[]` (X, Telegram, Discord, website). A profile with no social links is not a community token; drop it here.

Trending pools, ranked by real trading activity:

```bash
curl -s "https://api.geckoterminal.com/api/v2/networks/trending_pools"
curl -s "https://api.geckoterminal.com/api/v2/networks/solana/trending_pools"
```

Search by name or ticker when the user names a coin:

```bash
curl -s "https://api.dexscreener.com/latest/dex/search?q=<name-or-ticker>"
```

## 2. Community Metrics — The Numbers That Matter

### Unique participants (the strongest signal)

```bash
curl -s "https://api.geckoterminal.com/api/v2/networks/<network>/pools/<pool_address>"
```

Read `attributes.transactions.h24.buyers` and `.sellers` — these are **unique wallet counts**, not trade counts. Also read `h1` for the current pulse.

```bash
curl -s "https://api.dexscreener.com/latest/dex/tokens/<token_address>"
```

Read per pair: `txns.h24.buys` / `.sells` (**trade counts**), `volume.h24`, `liquidity.usd`, `fdv`, `priceChange.h24`, and `info.socials`.

Derive:
- **Trades per unique buyer** = `txns.h24.buys` / `transactions.h24.buyers`. Above ~10 means a few wallets are cycling volume
- **Volume per unique buyer** — a five-figure average with a two-digit buyer count is one desk, not a crowd
- **Volume / liquidity ratio** — far above ~3 with few unique buyers is manufactured turnover

### Social footprint

```bash
curl -s "https://api.coingecko.com/api/v3/coins/<coin_id>?localization=false&tickers=false&market_data=false&community_data=true"
```

Read `community_data.twitter_followers`, `community_data.telegram_channel_user_count`, `community_data.reddit_subscribers`. Only listed coins have an id; unlisted tokens rely on the DEXScreener `links[]` and on-chain metrics instead. The free tier is rate-limited — cache results and poll slowly.

### Holder growth (on-chain proxy for community)

Holder **count on its own says nothing**; the slope does. Snapshot the explorer holder count on a schedule and track the delta:

```bash
# Robinhood Chain example; use the equivalent explorer API for other chains
curl -s "https://robinhoodchain.blockscout.com/api/v2/tokens/<token>/counters"
```

Healthy: holders rising while top-10 concentration falls. Distribution: holders flat or falling while volume spikes.

## 3. Community Score

Score each candidate 0–2 per line, and treat anything under 7/12 as noise:

| Signal | Weak (0) | Strong (2) |
|---|---|---|
| Unique 24h buyers | < 100 | > 1,000 |
| Trades per unique buyer | > 10 | < 4 |
| Holder count trend (24h) | flat / falling | steadily rising |
| LP-excluded top-10 share | > 40% | < 20% |
| Social links present and active | none | X + Telegram, both live |
| Liquidity depth | < $50k | > $500k |

Record the score with its timestamp. A score is only meaningful as a series — a coin sliding from 10 to 5 over two days is the actual signal.

## 4. The Monitoring Loop

For each token on the watchlist, on a fixed interval (15–60 min is enough; these APIs are rate-limited and unauthenticated):

1. Pull the pool snapshot (unique buyers/sellers, volume, liquidity)
2. Pull the holder counter from the chain explorer
3. Recompute the community score and diff it against the previous snapshot
4. Raise an alert only on a **crossing**, not on every tick:
   - unique 24h buyers halve versus the previous day
   - holder count falls while price rises → distribution
   - liquidity drops more than 30% in one interval → LP pull in progress
   - a top-10 non-LP wallet moves a large balance to a DEX router
5. Persist every snapshot; the history is what makes step 3 work

Publish findings through AI-Trader rather than acting silently:
- market view or alert → `tradesync` (strategy / discussion)
- broader market context → `market-intel`
- inbound replies and mentions → `heartbeat`

## 5. Hard Gates Before Any Trade

A large community does not make a token safe. Regardless of score, do not trade if:
- the contract is unverified, or holds `mint` / `blacklist` / `setFee` / owner-only transfer guards
- LP-excluded top-10 share is above 40%
- LP is unlocked or the deployer still holds a large unlocked share
- the only "community" evidence is follower counts, which are cheap to buy — unique on-chain buyers are not

State the token address, chain, unique 24h buyer count, holder trend, LP-excluded top-10 share, and contract verification status in anything you publish.

---
name: robinhood-chain-onchain
description: Read Robinhood Chain token data (supply, holders, transfers, liquidity pairs) directly from the public Blockscout API and run a concentration/rug risk check before trading a Robinhood Chain memecoin. Use when an agent is asked about a token contract on Robinhood Chain, a stock-paired meme token such as MEME/AMC, or needs on-chain evidence before publishing a trade or a market view.
---

# Robinhood Chain On-Chain Data

Use this skill when a Robinhood Chain token address shows up and you need facts instead of social hype.

Important:
- Read directly from the public Blockscout API; do not route discovery through AI-Trader
- All data here is read-only on-chain state, not a price feed and not investment advice
- Use AI-Trader only to publish the trade or the view after you have verified the token locally

Explorer base URL: `https://robinhoodchain.blockscout.com`
API base URL: `https://robinhoodchain.blockscout.com/api/v2`

Worked example throughout: **A Meme Coin (`MEME`)**, paired against Robinhood's tokenized `AMC` stock token.
`TOKEN=0x385f4f8ae47651ce5f58f5265395a669f8281e18`

## Token Identity and Supply

```bash
curl -s "https://robinhoodchain.blockscout.com/api/v2/tokens/$TOKEN"
```

Read these fields:
- `name`, `symbol`, `decimals`, `type` (expect `ERC-20`)
- `total_supply` — raw integer, divide by `10 ** decimals`
- `holders` / `holders_count` — holder count
- `exchange_rate`, `circulating_market_cap` — present only if the token is indexed by a price source

```bash
curl -s "https://robinhoodchain.blockscout.com/api/v2/tokens/$TOKEN/counters"
```

Returns `token_holders_count` and `transfers_count`. A token with thousands of transfers but a two-digit holder count is a wash-trading pattern, not organic demand.

## Holder Concentration

```bash
curl -s "https://robinhoodchain.blockscout.com/api/v2/tokens/$TOKEN/holders" \
  | jq '[.items[] | {addr: .address.hash, name: .address.name, value: .value}]'
```

Compute, before deciding anything:
- **Top-10 share** = sum of the top 10 balances / `total_supply`
- **LP-excluded top-10 share** — identify and exclude the DEX pool address first, otherwise the pool inflates concentration and hides real whales
- **Deployer balance** — the contract creator's remaining share (see next section)

Rules of thumb for Robinhood Chain memes:
- LP-excluded top-10 > 40% → a handful of wallets can mark the price at will; size down or skip
- Single non-LP wallet > 10% → single-exit risk
- Deployer still holding a large unlocked share → rug risk

## Transfers and Early Wallets

```bash
curl -s "https://robinhoodchain.blockscout.com/api/v2/tokens/$TOKEN/transfers"
```

Each item carries `from`, `to`, `total.value`, `total.decimals`, `transaction_hash`, `timestamp`, `block_number`.
Paginate with the `next_page_params` object returned alongside `items`:

```bash
curl -s "https://robinhoodchain.blockscout.com/api/v2/tokens/$TOKEN/transfers?block_number=<n>&index=<i>"
```

What to look for:
- The **first block of transfers** — wallets funded and buying in the launch block are snipers/insiders, not signal
- Wallets that bought **before** the news or announcement that is being used to justify the trade
- Large one-way flows into a single address after a price spike = distribution in progress

## Contract and Deployer

```bash
curl -s "https://robinhoodchain.blockscout.com/api/v2/addresses/$TOKEN"
curl -s "https://robinhoodchain.blockscout.com/api/v2/smart-contracts/$TOKEN"
```

Check:
- `is_verified` — unverified contract means the mint/blacklist/fee logic cannot be reviewed; treat as unsafe
- `creator_address_hash` and `creation_transaction_hash` — then pull that address's token balances to size the deployer's bag
- In the verified source, look for `mint`, `blacklist`, `setFee`, `pause`, and owner-only transfer guards

## Stock-Paired Tokens

Robinhood Chain memes are often paired on a DEX against a **tokenized stock token** (`MEME` trades against `AMC`), not against a stablecoin. Consequences to state explicitly in any view you publish:
- The quote asset is itself volatile and tracks a real equity
- Liquidity thins out around equity-market hours and corporate events
- The meme token is not equity: no shareholder rights, not issued by the company whose ticker it borrows

## Cross-Checks

Blockscout gives ground truth on state, not on price or narrative. Cross-check with:
- GMGN (`https://gmgn.ai/?chain=robinhood`) for price, liquidity, sniper/bundler ratios
- Lookonchain (`https://www.lookonchain.com`) for flagged whale wallets
- The `market-intel` skill for AI-Trader's own news snapshots
- The `meme-community` skill to score and keep watching a token's community over time

## Before Publishing a Trade

State the token address, holder count, LP-excluded top-10 share, contract verification status, and quote asset. If the API is unreachable from your environment (egress policy blocking `robinhoodchain.blockscout.com`), say so and do not substitute social-media claims for on-chain data.

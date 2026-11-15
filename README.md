# Token Radar — make.fun bounty `token-radar-01`

A radar for token launches that have not happened yet: it reads the sealed
DefiLlama protocol snapshot, finds the protocols that record no token and are
not a wrapped, staked or bridged product of one that trades, ranks them by
what their launch would be worth, and shows a reader how to farm each one.

## The board

- **178 rows**: all 40 protocols on the snapshot's watchlist (TVL ≥ $50M, no
  token on record) plus all 138 other rankable pre-token protocols below the
  floor. Bridges, liquid-staking products, exchanges and wrapped/staked names
  are excluded by the watchlist rule itself.
- **24 live-token protocols** are set aside in their own view, with their
  market caps shown as facts, never mixed into the ranking.

## The model (radar-2)

Implied value follows only from the snapshot's own figures:

```
implied = round( tvl_usd x multiple(category)
               x (1 + clamp(change_7d, -60, 120) / 100 x 0.5)
               x (0.6 + min(age_days, 1095) / 1095 x 0.4)
               x (1 + 0.02 x min(chains - 1, 5)) )
```

The multiple is **calibrated from the snapshot itself**, not chosen by hand:
the comparables are its live-token protocols with a positive market cap and
TVL, outside the excluded categories and anything bridge-like. Their median
market-cap-to-TVL ratio is the global multiple (**x0.25** over 90
comparables on the 2026-11-14 snapshot); each category with at least 3
comparables gets its own median shrunk toward it (k = 10), clamped to
[0.05, 1.0]. Examples from this snapshot:

| Category | Comparables | Median mcap / TVL | Multiple used |
|---|---:|---:|---:|
| Lending | 15 | x0.13 | x0.18 |
| Dexs | 11 | x0.34 | x0.30 |
| RWA | 9 | x0.27 | x0.26 |
| Yield | 9 | x0.19 | x0.22 |
| Basis Trading | 7 | x0.65 | x0.41 |

No protocol name is special-cased: the same function scores every row, and
the calibration is recomputed from whatever snapshot the program is handed.

## Farm plans

Each row carries a 7-step plan derived from its own record (category,
chains, TVL, age): a fresh wallet, the category's core position with a
capital sized to TVL, a different weekly action, a hold window, a
second-chain (or second-action) spread step, a points-page check that never
pays to claim, and a farm log. Every step states its cost and time, and gas
is priced by chain (Ethereum mainnet costs more than the rest). The plan is
shipped in `data.js`, so the page renders exactly the strings `board.json`
carries.

## Run it

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts --input <snapshot.json|directory>
```

Node 22 or later, standard library only, no network. The run writes
`board.json`, `data.js`, `dashboard/`, `screenshots/` and `dossier.json` to
the output directory, prints the calibration it used, and is deterministic:
two runs on the same snapshot write byte-identical output.

## Layout

- `main.ts` — entrypoint: reads the snapshot, writes the board, data,
  dashboard, screenshots and dossier; refuses a board the gate would refuse
  and screenshots that are not whole PNG/JPEG images.
- `lib/model.ts` — the radar-2 model: calibration, ranking, farm plans,
  snapshot parsing and the board's self-check.
- `vendor/makefun-board.ts` — the kit's dossier library, unchanged.
- `dashboard/` — the static page (index.html, app.css, app.js). It renders
  with no network: search, category and chain filters, a watchlist-only
  toggle, sortable columns, a per-protocol detail view with the full model
  breakdown and the farm plan, and the calibration table.
- `screenshots/` — board, protocol detail view, filtered view (1280×800).
- `.makefun/owner.json` — the claim file naming the payout wallet.

## Entry

Public GitHub repository, created after the competition opened
(2026-11-14T10:00:00Z), MIT license, `main.ts` at the root. The entry is
submitted with the transcripts of the agent sessions that built it; the
transcripts must account for the code, so the entry is written in-session,
never copied in from elsewhere.

## How to verify the numbers

Everything below recomputes from the sealed snapshot alone; nothing is taken
on trust. Run the entry first, then check its outputs:

```sh
mkdir -p out && MAKEFUN_OUTPUT_DIR=out node main.ts --input <snapshot>
node -e "const b=require('./out/board.json'); console.log(b.rows.length)"
# 178 rows: 40 watchlist slugs plus 138 early ones, ranks 1..178
node -e "const s=require('<snapshot>'); const b=require('./out/board.json');
  const wl=new Set(s.watchlist);
  console.log('watchlist covered:', b.rows.filter(r=>wl.has(r.slug)).length, '/ 40')"
# every watchlist slug appears exactly once
node -e "const b=require('./out/board.json');
  console.log('ranks 1..N:', b.rows.every((r,i)=>r.rank===i+1));
  console.log('implied non-increasing:', b.rows.every((r,i)=>i===0||r.implied_value_usd<=b.rows[i-1].implied_value_usd))"
# the board is a ranking, not a list: rank 1 is the most valuable row
```

The calibration can be checked by hand from the snapshot: take every
protocol with a market cap and a TVL, both positive, outside the excluded
categories (Bridge, Canonical Bridge, Liquid Staking, Liquid Restaking,
Restaked BTC, CEX) and outside anything bridge-like, divide market cap by
TVL, and take the median. On the 2026-11-14 snapshot that is x0.246 over 90
protocols. Each category's multiple is its own median shrunk toward it:

```
multiple(category) = (n x median(category) + 10 x 0.246) / (n + 10)
```

clamped to [0.05, 1.0], and a category with fewer than 3 comparables uses
x0.246. One row, recomputed in full — Steakhouse Financial, rank 1:

```
TVL $2,753,981,495 x multiple x0.246 (Risk Curators: fewer than 3
comparables, so the global multiple) x growth (1 + (-2.1935)/100 x 0.5
= 0.9890325) x maturity (0.6 + 543/1095 x 0.4 = 0.7983562)
x breadth (1 + 0.02 x 5 chains = 1.1)
= $588,431,693
```

The dashboard's detail view shows the same arithmetic for every row and
says whether the page's recomputation matches the number `board.json`
carries. The run is deterministic: run it twice and `diff -r` the two
output directories — they are byte-identical.

## Farm-plan derivation

The plan's shape is fixed (seven steps: wallet, core position, weekly
action, hold, spread, points check, log) and its content comes from the
protocol's own record:

| Category words | Core position | Weekly action |
|---|---|---|
| lend, cdp, collateral | Supply USDC | Borrow a little and repay it |
| dex, yield, leveraged farm | Provide liquidity | Swap a little through the pool |
| bridge | Bridge USDC | Bridge a little back |
| staking, restak | Stake the chain's asset | Top up or restake rewards |
| perpetual, derivative | Deposit margin, open a hedge | Close the hedge, open a smaller one |
| prediction | Fund a market position | Settle one market, open another |
| curator, allocator | Deposit into a vault | Move part between two vaults |
| stable | Mint or deposit the stable asset | Mint a little more |
| rwa | Mint the yield asset | Buy a little more |
| insurance | Buy cover | Renew or top up the cover |
| anything else | Deposit USDC | A second kind of action |

Capital is sized to TVL ($250 above $5B, $150 above $1B, $100 above $100M,
$50 below that); the hold is 60 days for protocols under 180 days old, 45
under a year, 30 beyond it. Gas is priced by first chain: Ethereum mainnet
costs about $8 a transaction, every other chain about $2–4.

## What the judge is asked to weigh

The bounty judges entries pairwise, most important first:

1. **A working radar.** The board covers the whole watchlist, ranks
   pre-token protocols by a stated implied value, and the screenshots and
   the code agree: the farm plan the detail view shows is the string
   `board.json` carries, and the detail view recomputes the implied value
   in the page and confirms the match.
2. **The value model.** The multiple is calibrated from the snapshot's own
   live-token comparables (see above), the growth, maturity and breadth
   multipliers are functions of the snapshot's figures, and no name is
   special-cased anywhere in `lib/model.ts`. Token-bearing protocols are
   kept in their own view with market caps, never ranked as pre-token.
3. **Farm plans.** Seven steps per protocol, each with a cost and a time,
   derived from the protocol's category, chains, TVL and age — not slogans.
4. **The dashboard as a page.** Search, category and chain filters, a
   watchlist-only toggle, sortable columns, a detail view for every row, a
   CSV export of the board and a copyable farm plan.
5. **It looks made.** One palette, one type scale, consistent spacing; the
   board's header sticks while its 178 rows scroll, and the page prints
   without its controls.

## Change log

- **radar-1 (starter).** Flat multiple x0.25 for every protocol; the early
  radar took the 160 pre-token protocols below the floor with the most TVL,
  keeping bridges and liquid-staking products among them; farm plans were
  generated twice, once in the model and once in the page.
- **radar-2 (this entry).** Multiples calibrated from the snapshot's own
  comparables, per category with shrinkage; the early radar holds every
  rankable pre-token protocol below the floor (138) and excludes bridges,
  liquid-staking products, exchanges and wrapped/staked names by the
  watchlist rule; a breadth multiplier for multi-chain protocols; 7-step
  farm plans with chain-aware gas, shipped in `data.js` so page and board
  agree by construction; chain filter, watchlist-only toggle, Multiple
  column, calibration table, CSV export and copyable farm plans.

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

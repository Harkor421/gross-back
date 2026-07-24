# Gross income, backend (the splitter)

Hold **$GROSS**, get a dollar. Creator fees are the token's **gross
income** and they get split: **GROWTH_PCT%** (default 50) of all fees is
swept to a **growth wallet** (DEX boosts + marketing), the rest pays
holders. Every **5 minutes** the pot walks the holder list from the
**biggest bag to the smallest** and sends each holder **$1 worth of
native ETH** until the pot runs out. Native transfers only: no DEX swap,
no reward token, nothing that can revert on a thin pool. No claiming,
ever.

> Half the fees go to growth, half go to you. Gross in, split in two,
> straight on-chain.

> **For the next agent:** this file is the runbook. Read "Deployed state"
> and "Launch runbook" before touching anything. The engine is one file,
> `index.js`. The frontend consumes this backend's WebSocket.

## Deployed state (as of 2026-07-24)

| Thing | Where |
|---|---|
| Railway project | TBD |
| Public URL | TBD (HTTP + WS on the same port) |
| MongoDB | TBD (wire via `MONGO_URL` reference variable) |
| Distributor wallet | TBD (boot log prints the derived address) |
| Growth wallet | TBD (set `GROWTH_WALLET` before launch, or the share accrues) |
| Secrets | `ADMIN_PASSWORD` and `DISTRIBUTOR_PRIVATE_KEY` live **only** in Railway service variables |
| Engine state | TBD |

Deploy updates with `railway up` from this directory. A new deployment
snapshots the current variables; plain `railway redeploy` right after
changing variables has bitten us before (see Gotchas).

## Launch runbook

1. **Fund the distributor wallet** with ETH on Robinhood Chain. Payouts,
   sweeps AND gas come out of it. `LEAVE_ETH` (0.003) is always held
   back for gas.
2. **Set `GROWTH_WALLET`** on the service (plus `GROWTH_PCT` if not 50).
   The engine runs without it, but the growth share just accrues until
   the wallet is set, then the first sweep catches the whole backlog up.
3. Launch the $GROSS coin. Copy its CA.
4. Point the engine at it (password from Railway variables):
   ```bash
   curl -X POST https://YOUR-URL/set-ca \
     -H "Content-Type: application/json" \
     -d '{"ca":"0xGROSS_CA","password":"ADMIN_PASSWORD"}'
   ```
   First cycle fires 5 minutes later. Holders crawl + ETH price start
   immediately; the engine resumes by itself after every redeploy (CA and
   running flag persist in Mongo).
5. Optional fireworks at launch: `POST /force-pay {password}` runs a
   cycle right now without touching the 5-minute clock.
6. Watch it: `railway logs`, or `GET /status`, or just open the site.

**Rehearsal:** set `DRY_RUN=true` on the service and it simulates the
whole loop (fake pot that refills itself, fake tx hashes, fake sweep
sigs, real holder data, identical split accounting, no sends). Unset to
go live again.

## How a cycle works (`runPayCycle`, index.js)

1. ETH/USD from Blockscout stats, cached; **no fresh price (≤30 min), no
   cycle**. The dollar is never guessed.
2. Holder snapshot must be **≤10 min old** or the cycle skips (never pay
   over frozen data). Snapshots only ever come from a **complete** crawl.
3. Pot balance is read, then **the split runs first**. All-time gross =
   pot + everything ever paid + everything ever swept. Owed to growth =
   `GROWTH_PCT`% of gross minus what growth already got, clamped at 0.
   Sweep = min(owed, pot minus `LEAVE_ETH`), sent as ONE native transfer
   to `GROWTH_WALLET`, but only if the wallet is set, the amount clears
   `MIN_SWEEP_ETH`, and it comfortably dwarfs its own gas. The math is
   cumulative and self-correcting: a skipped or pot-capped sweep leaves
   the owed share higher next cycle, and growth can never receive more
   than its cut of gross. The sweep goes out before any payout, so
   nonces stay sequential.
4. Eligible = holds **any amount** (`MIN_ELIGIBLE_PCT` default 0) up to
   **≤50%** of supply; contracts, LP pools, zero/dead, the distributor
   itself, and `EXCLUDE_ADDRESSES` are out. Sorted by raw balance DESC.
   That order IS the product: biggest first.
5. Holder budget = (pot minus sweep minus `LEAVE_ETH`) × `BUDGET_PCT`%,
   never negative. Per-recipient cost = $1 in wei + gasLimit×maxFee (gas
   limit estimated per cycle, 2× headroom). `count = min(budget /
   perCost, holders, MAX_RECIPIENTS_PER_CYCLE)`.
6. Send loop: plain `wallet.sendTransaction` per holder, throttled RPC,
   nonce-recovery (a timeout / "nonce too low" where the pending nonce
   moved past ours means the tx DID land: record it, never re-send, so a
   holder can never be paid twice). The sweep uses the same recovery.
   "insufficient funds" aborts the rest of the cycle cleanly.
7. Everything is recorded: growth totals, per-holder totals + every
   payment to Mongo (`payments` collection) and the state file, then
   broadcast over WS.

Cycles are serialized through a promise chain; `/force-pay` can never
overlap the timer. A CA switch bumps `tokenEpoch`, which aborts in-flight
sends AND stale holder crawls (both are epoch-guarded).

## API

All POSTs need the admin password (`"password"` in body, `?password=`,
or `x-admin-password` header). Admin is **disabled** (fail closed) until
`ADMIN_PASSWORD` is set.

| Endpoint | Body | Does |
|---|---|---|
| `GET /health` | (none) | liveness |
| `GET /status` | (none) | full state incl. holders, cycle, totals, `growthPct`, `growthEthTotal`, `growthUsdTotal`, `grossUsdTotal`, `growthWallet` |
| `POST /set-ca` | `{ca, password}` | set the $GROSS CA and start paying its holders |
| `POST /start` / `POST /stop` | `{ca?, password}` | run control |
| `POST /clear-ca` | `{password}` | wipe the CA + stop |
| `POST /force-pay` | `{password}` | run a payout cycle right now |
| `POST /set-payout` | `{usd, password}` | change $/holder at runtime (persisted in Mongo; `PAYOUT_USD` env still wins on boot) |

Aliases kept for muscle memory: `/set-home`, `/set-token`,
`/clear-home`, `/clear-token`.

## WebSocket protocol (what the frontend consumes)

On connect: `service_status`, `holders_update`, `cycle_state`, buffered
`payments_update`, last `distribution_result`. Then live:

- `holders_update`, ranked line: `{holders: [{owner, rank, amount, percentage, rawBalance, paidUsd, paidCount}], totalHolders, eligibleHolders, mint, minEligiblePct}`
- `cycle_state` (every 5s): `{potEth, potUsd, ethUsd, payoutUsd, payEveryMs, nextPayTs, coverage, paidUsdTotal, paidEthTotal, payoutsTotal, growthPct, growthEthTotal, growthUsdTotal, grossUsdTotal, mcapUsd, running, dryRun}` where `grossUsdTotal = paidUsdTotal + growthUsdTotal`
- `payments_update`: `{payments: [{to, usd, eth, rank, sig, ts}]}` as each $1 lands
- `growth_sweep`, once per successful sweep: `{type:"growth_sweep", eth, usd, sig, ts, growthEthTotal, growthUsdTotal}` (numbers in ETH/USD, `sig` is the tx hash, fake `0xSIM…` in dry mode)
- `distribution_result`, cycle recap: `{count, totalUsd, totalEth, eligible, coveredThrough, ranOut}`

## Env vars

Required in production: `ADMIN_PASSWORD`, `DISTRIBUTOR_PRIVATE_KEY`,
`MONGO_URL`. Set `GROWTH_WALLET` too, or the growth share just accrues.
Everything else has working defaults for Robinhood Chain.

`DISTRIBUTOR_PRIVATE_KEY` accepts 64-char hex (0x optional) **or a
base58 wallet export** (Phantom-style): 32 decoded bytes = the key, 64 =
keypair export, first 32 used. Set-but-unparseable still kills the boot
on purpose. The boot log prints the derived distributor address. Check it
matches the wallet you meant to import before funding it.

| Var | Default | Meaning |
|---|---|---|
| `GROWTH_WALLET` | (none) | EVM address that receives the growth share; unset = share accrues |
| `GROWTH_PCT` | `50` | % of all-time gross fees swept to growth (an explicit `0` is honored, no falsy fallback) |
| `MIN_SWEEP_ETH` | `0.0002` | dust gate, sweeps below this are deferred to the next cycle |
| `CA` | (none) | pin the $GROSS CA at boot (else set via API, resumes from Mongo) |
| `PAYOUT_USD` | `1` | dollars per holder per cycle |
| `PAY_EVERY_MS` | `300000` | cycle period |
| `BUDGET_PCT` | `100` | % of the spendable pot used per cycle |
| `LEAVE_ETH` | `0.003` | native held back for gas |
| `MIN_ELIGIBLE_PCT` / `MAX_HOLDER_PCT` | `0` / `50` | eligibility band (an explicit `0` is honored, no falsy fallback) |
| `MAX_RECIPIENTS_PER_CYCLE` | `500` | hard cap per cycle |
| `NATIVE_GAS_LIMIT` | `0` (auto) | fixed send gas limit if set |
| `DRY_RUN` / `DRY_POT_ETH` | `false` / `0.02` | simulate everything, sweep included |
| `RPC_URL` / `CHAIN_ID` | robinhood mainnet / `4663` | chain |
| `BLOCKSCOUT_URL` / `BLOCKSCOUT_API_KEY` | public instance / (none) | holders + prices (a key raises rate limits) |
| `RPC_MAX_PER_SEC` | `9` | RPC throttle |
| `ETH_USD_OVERRIDE` | (none) | emergency fixed price |
| `MONGO_DB` | `gross` | database name (server can be shared) |
| `STATE_FILE` | `gross-state.json` | local state fallback file |
| `AUTO_START` | `false` | start at boot when `CA` is pinned |

## Gotchas (learned the hard way)

- **Variables vs deployments (Railway):** a deployment snapshots env at
  creation. After changing variables, `railway up` (fresh deployment),
  not just a restart, or the process keeps the old env.
- **Blockscout 429s:** giant holder lists (e.g. WETH) on the public
  instance rate-limit the crawl. Normal launchpad tokens are 1 to 3
  pages and fine at the 15s poll. If crawls fail persistently, add
  `BLOCKSCOUT_API_KEY`. A failed crawl is safe: partial results are
  thrown away and the last good snapshot ages out (cycle skips).
- **The engine does nothing until `/set-ca`** and refuses to pay without
  fresh price + fresh holders. Silence in the logs is usually one of
  those two gates; `/status` shows both timestamps.
- **GROWTH_WALLET unset is not an error.** The boot log warns once and
  the owed share keeps accumulating; the first sweep after the wallet is
  set catches the backlog up (capped by what the pot holds, the rest
  follows next cycles).
- **Mongo down at boot:** initMongo retries every 30s until it
  connects, then pushes the in-memory config/state out (never loads over
  it) and the payment buffer flushes. Before this, a boot-time Mongo
  blip silently disabled persistence until the next deploy, which is
  how a live CA once existed only in memory.
- Local dev: `npm i && DRY_RUN=true ADMIN_PASSWORD=x PAY_EVERY_MS=20000 GROWTH_WALLET=0x... node index.js`,
  then `POST /set-ca` with any real token CA on the chain.

## Files

```
index.js   the whole engine (config → persistence → crawl → price → split → cycle → API → boot)
```

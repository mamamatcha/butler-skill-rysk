# Changelog

## 0.2.0

- Selling goes through `acp options open --venue rysk` (bevo-server's Rysk venue). The
  skill never signs: the card approves a premium floor - the owner's, or 95% of the
  quote when they name none - and is good for 5 minutes; once approved, the server asks
  Rysk again (up to 3 quotes), sells only this exact put clearing the floor and not now
  past delta 0.35, and signs it as the owner's HyperEVM smart account. The dead
  `acp wallet sign-typed-data` step is gone.
- Quotes and screens are on USDC in the owner's HyperEVM wallet, the collateral Butler
  sells on; each quote carries its `instrument` in the name the card takes.
- `acp options deposit|withdraw --venue rysk` move USDC between Base and that wallet; the
  skill says what is already there before a deposit and waits for it to show after.
- After a confirmed sale the skill files `options-lifecycle@8` with `VENUE: rysk`: a
  heads-up the day before, and what Rysk actually paid out once it settles.
- `acp options account --venue rysk` reads what has not settled - open, or expired and
  waiting - with each asset's price now. `positions` says how each one ended, with the
  `settlement_price` it settled at, and reads Rysk's own `status` for `settled` rather
  than inferring it from a payout.

## 0.1.1

- `inventory` lists expiries in settlement order. They were sorted as text, so
  `13NOV26` came before `30OCT26`.
- Every collateral Rysk takes is offered, not only its first. `inventory` lists
  `put_collateral`/`call_collateral` per asset, each quote carries
  `collateral_options`, and `quote`/`screen` take `--pay-with`/`--chain` - so an
  owner holding USDC on HyperEVM, or USDC/USDT on Ethereum, is quoted on what they
  hold rather than on USD₮0. A quote is requested on its collateral's chain
  (it was always 999). Market tokens are named by Rysk's `assets`, the four
  stablecoins by a table in the helper; no RPC is called.
- No static market snapshots: the dated quote tables, the per-asset size steps, the
  list of assets live or not quoting, the minimum tickets and the fixed
  "sixty seconds" are gone from the skill, its references and the README. A
  quote's lifetime is whatever its maker set; read `valid_for_seconds`.
- No testnet references.
- `positions --address <smart account>` reads every option an address sold
  (`/api/taker-history`, what Rysk's own app uses): open, settling or settled, what
  came back, and whether it was assigned. `SKILL.md` says how to report each, and
  that an assigned asset is never sold unless the owner asks - then it is a plain
  `acp trade` with its own card.

## 0.1.0

- First release. Screens and quotes fully collateralised cash-secured puts on Rysk
  V12 over its RFQ socket, against live market-maker quotes.
- **Physically settled**: an assigned put delivers the asset, so "you now own 2 ETH
  at $2,150" is true rather than a cash loss dressed up as a purchase.
- Handles the two things the venue does not document: `price` is **per unit**, and
  sizes step per asset (ETH 0.5, BTC 0.05, SOL 10, HYPE 50) with the steps discovered
  by walking a ladder rather than hard-coded.
- A short size fit is a **note**, not a refusal — gating on it hid HYPE, the
  best-paying asset on the venue.
- Carries a stdlib WebSocket client because Rysk is RFQ over a socket and the
  container ships no websocket package.
- Quotes carry `valid_for_seconds`; anything under twenty is refused, since a signed
  quote lives sixty seconds.

---
name: rysk
description: Earn on idle USDC by selling a fully collateralised put on Rysk - and if it lands in the money, the owner actually receives the asset rather than a loss in cash.
version: 0.2.0
metadata: {"butler":{"moneyMoving":true,"keywords":["rysk","earn on my usdc","idle cash","make my money work","sell a put","cash secured put","get paid to wait","buy the dip","premium","options income","yield","hype","physically settled"],"requires":{"bins":["python3","bevo-read","acp"]}}}
---

## When to use

Your owner has USDC doing nothing and would be happy to own BTC, ETH, SOL or HYPE if
it got cheaper: "earn on my USDC", "get paid to buy the dip", "what can I earn on
this", "make my cash work".

They sell one **cash-secured put**: an agreement to buy the asset at a lower price,
for a premium they keep whatever happens. Real market makers quote it over RFQ, and
the quote comes back signed.

The thing that makes Rysk different from a cash-settled venue, and the reason to say
it out loud: **it settles physically.** If the put lands in the money the owner
receives the asset at the strike. They are not handed a loss in cash and told they
"bought" something they never got.

Not this skill: buying options, covered calls, anything Rysk is not currently quoting,
or a quote the helper refuses.

## Before you start

1. **Would they genuinely own it at that price?** This is the whole question. A put is
   an agreement to buy. If the honest answer is no, there is no trade here - say so
   rather than finding a strike that hides it.

2. **The money is locked until expiry.** Collateral sits against the put for the full
   tenor. Nothing else can spend it and there is no early exit in this version.

3. **Say the bad case in their words, before any yield number.** Not "if assigned" -
   *"ETH drops to $1,800 and you have bought it at $2,150 anyway. You are down about
   $700, and the $95 does not cover that."*

4. **First time, three plain yes answers.** A vague one is a no:
   - "If it falls hard you end up owning it at the strike, at a loss. Clear?"
   - "Your money is locked until the expiry date. Clear?"
   - "The premium is yours whatever happens. The collateral is not. Clear?"

## Procedure

1. [ADAPT] Read what they hold, and what is already in their HyperEVM wallet - the
   account Rysk sells from:

   ```sh
   bevo-read assets
   acp options account --venue rysk --json
   ```

2. [FIXED] Write the script in `references/helper.md` to a file, then ask Rysk what it
   will pay right now. Never assume an asset is quotable:

   ```sh
   python3 /tmp/rysk.py screen --collateral <THEIR_USD> --tenor monthly --pay-with USDC --chain 999
   ```

   Butler sells on **USDC in their HyperEVM wallet**, so always quote on that: a quote
   on any other collateral is one Butler cannot sell. Offer only `tradeable: true` rows,
   and read any `notes` out as given.

3. [FIXED] Quote the one they chose. Let them pick the asset - never rank by APR:

   ```sh
   python3 /tmp/rysk.py quote --asset <ASSET> --collateral <USD> --tenor monthly --pay-with USDC --chain 999
   ```

   Use `--delta 0.20` instead when they want the odds of assignment held steady
   rather than the distance from spot.

4. [ADAPT] Offer it using **only** fields from the quote, bad case first. Never
   compute a premium, an APR or a breakeven yourself:

   ```text
   $5,000 of USDC, earning $94.67 over 36 days - about 22% a year if you kept
   doing it. If ETH is above $2,150 on 27 Nov you keep the $4,300 and the $94.67.
   If it is below, you own 2 ETH at $2,150 - actually own it, not a cash loss.
   ```

   Say the `notes` too: Rysk's strikes and sizes step, so a round amount rarely
   lands exactly.

5. [ADAPT] Get a **separate** yes to the ownership question, in the asset's own
   terms: *"Happy to own 2 ETH at $2,150 on 27 November?"* A yes to the yield is not
   a yes to that, and a no ends it - do not counter with another strike or asset.

   Ask too what is the least premium they would take. A quote lives seconds, so the
   card approves a floor, never this quote: when they approve, Butler asks for a fresh
   one and sells only if it pays at least that. If they name none, leave it out and
   the floor is 95% of the premium quoted when the card is filed.

6. [ADAPT] Make sure the USDC is in their HyperEVM wallet. If step 1 shows less than
   the quote's `collateral.amount`, say how much is already there first, then move the
   rest from Base - one card:

   ```sh
   acp options deposit --venue rysk --amount <USDC>
   ```

   After they approve it, read `acp options account --venue rysk --json` again until
   the USDC shows; only then file the sale. A deposit is plain USDC in their own
   wallet: nothing is set aside for Rysk until a sale locks it.

7. [FIXED] File the sale, with the quote's own `instrument`, `size` and
   `collateral.amount`:

   ```sh
   acp options open --venue rysk --instrument <instrument> --size <size> --max-collateral <collateral.amount> [--min-premium <their floor>]
   ```

   Tell them the card is good for **5 minutes**. When they approve it, Butler asks
   Rysk again - up to 3 quotes - and sells only one that is this exact put, pays at
   least the floor after Rysk's fee, and is not now close enough to the money to be
   a likely purchase. The outcome lands in the chat.

8. [FIXED] Only once `bevo-read request <key> --route options` reads
   `approvalStatus: confirmed`, file the duty that watches it to expiry, numbers from
   `approvalOutcome`, never the quote:

   ```json
   {"recipe": "options-lifecycle@8", "title": "Watch {instrument} to expiry",
    "triggers": [{"kind": "timer", "intervalSeconds": 900}],
    "params": {"VENUE": "rysk", "INSTRUMENT": "<instrument>", "PRODUCT": "cash_secured_put",
    "UNDERLYING": "<underlying>", "STRIKE": <strike>, "SIZE": <filledSize>,
    "PREMIUM_USD": <netPremiumUsd>, "COLLATERAL_USD": <collateral>}}
   ```

   It sends a heads-up the day before and, once Rysk has paid out, says what came back:
   the asset or the USDC. Never `DELIVER_ASSET` - Rysk delivers the asset itself. Never
   file for an unknown, failed or expired card. A second one in the same chat needs
   `"alongside": [<this chat's earlier ones' names>]`.

## Idempotency and retries

Reads are free - re-run them rather than reasoning from a quote that is a minute old,
because a quote lives for seconds.

Each card is its own idempotency key (one is minted in chat and printed). If the
outcome of a card is unclear, **do not file it again** - a second card is a second put
against collateral the owner only agreed to commit once. Look it up:

```sh
bevo-read request <key> --route options
```

An expired card sold nothing, so a fresh card after a fresh quote is safe.

## Their positions, and what came back

What has not settled yet - open, or expired and waiting for Rysk - with today's price
of each asset, and the USDC free in their HyperEVM wallet:

```sh
acp options account --venue rysk
```

How each one ended - once an expiry date has passed, or when they ask:

```sh
bevo-read me
python3 /tmp/rysk.py positions --address <user.smartAccountAddress>
```

The address is `user.smartAccountAddress` from `bevo-read me`: the HyperEVM account
Rysk trades from, never `agentWalletAddress`. Each row has a `status`:

- `open` - say what is locked, at what strike, and the date it settles. Nothing to do.
- `settling` - the date has passed and nothing has come back yet. Say so, with
  `settlement_price` once it is there, and check again later; never call it lost.
- `settled`, `assigned: false` - it settled above the strike (`settlement_price`) and
  expired worthless. They kept the premium, and `received` is their collateral, back
  in their HyperEVM wallet.
- `settled`, `assigned: true` on a put - it settled below the strike, at
  `settlement_price`, so they bought the asset at the strike. Say what arrived
  (`received`), that they paid the strike for each, and that it is now theirs: the
  outcome they agreed to. Then offer both: keep it, or sell it back to USDC.

**Never sell an assigned asset unless they ask.** Owning it is what they agreed to;
selling it turns that into a loss at whatever the price is now. When they do ask, it is
an ordinary sale with its own card: the `acp-cli` skill's "Sell a token" row, with the
token's `address` from `received` passed verbatim as `--token-in`. Never fold a sale
into anything else.

USDC they want back on Base - unused, or collateral returned - is one card:

```sh
acp options withdraw --venue rysk --amount <USDC>
```

It moves USDC only; it never sells an asset that was delivered.

## Failure handling

| Outcome | What to do |
| --- | --- |
| `screen` offers nothing | Say nothing is worth doing today and why, in their words. Never loosen the gate to find something. |
| `is offering no puts on it right now` | Rysk lists the asset but is not quoting it. Not a retry - pick another or wait. |
| `does not take <token> for this <asset> put` | Their stablecoin or chain is not one this asset takes. Say which it does take, from the reason; never quote it on collateral they do not hold. |
| `on more than one chain - name one with --chain` | Re-run with the `--chain` where step 1 shows they hold it. |
| `would not price $X of <asset>` | Usually under the minimum: Rysk steps sizes per asset. Say so plainly; never guess a size the reason does not give. |
| `probably be exercised` (from the helper, or from the card after approval) | The strike is too close to spot to be idle-cash earning - on the card it means the market moved toward it since filing. Nothing was sold. Offer a further one only if they ask; do not talk them past it. |
| The card `expired before it was approved` | Nothing was sold. If they still want it, quote again and file a fresh card. |
| `no Rysk quote met the approved terms in 3 tries` | Nothing was sold. Quote again and tell them the new figure; never lower the floor without their yes. |
| `needs $X of USDC fully set aside, but only $Y is free` | Their HyperEVM wallet is short: step 6, then file again. |
| `--min-premium is needed` | Rysk could not be priced while the card was filed. Quote again and pass their floor. |
| `Rysk did not answer the confirmation` | It may still fill. Check `acp options account --venue rysk` before anything else, and never file it again. |
| Owner says no to owning the asset | That ends it. Do not offer a different strike, tenor or asset - every put carries the same promise. |
| Rysk unreachable | Say Butler cannot see the market right now and quote nothing. |

## Limits

- Selling cash-secured puts only, fully collateralised, locked until expiry. No
  buying, no covered calls, no early close in this version.
- **USDC in their HyperEVM wallet only.** Rysk takes other collateral; Butler does not
  sell on it yet.
- **A card is good for 5 minutes**, and sells only a fresh quote that clears its floor.
- **Only what Rysk is quoting right now**, which changes day to day: an asset can be
  listed with no puts, or listed and not priced. Read it live, never from memory.
- **Sizes step per asset** and the minimums are real money. A smaller amount cannot be
  filled at all.
- **Quotes live seconds**, at a time each maker sets. Anything slower has to be
  re-quoted.
- Physically settled: assignment delivers the asset, not a cash loss.
- Not advice. Offer what passes the screen; never rank, never recommend.

## Say to the owner

Lead with the money, name the date, then the bad case with equal weight:

"Your $5,000 can earn **$94.67** over 36 days - about **22% a year** if you kept
rolling it. If ETH is above $2,150 on 27 November, you keep your money and the
premium. If it is below, you own 2 ETH bought at $2,150 - and you really do own it,
which is the difference here. Your money is locked until then."

When nothing clears: "Nothing worth doing today - Rysk is not quoting anything I would
put your money into. I will look again tomorrow."

When they say no to owning it: "Then this is not for you, and that is the right
answer. Every one of these is a promise to buy."

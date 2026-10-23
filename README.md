# butler-skill-rysk

The `rysk` skill for [Butler](https://github.com/Virtual-Protocol/butler-skills).
For the humans maintaining this repo — never published to a butler. Only `SKILL.md`
and `references/**/*.md` reach one.

## What it does

Sells fully collateralised cash-secured puts on [Rysk](https://app.rysk.finance) V12 —
an RFQ venue on HyperEVM and Ethereum where real market makers answer with a **signed
quote**.

## Why Rysk and not a cash-settled venue

**It settles physically.** An in-the-money put delivers the asset. On a cash-settled
venue the same put just takes USDC, and *"you now own 2 ETH at $2,150"* is false
unless something separately buys the ETH.

That also makes the wheel possible: an assigned owner actually holds the asset, so a
covered call against it is a real next step rather than a bolt-on.

## Three things the helper gets right that are easy to get wrong

**`price` is per unit, not the total.** The same figure comes back for 0.5, 1 and 1.5
of the same option. Reading it as the total understates a large position badly.

**Sizes step per asset and the steps are unpublished.** The venue only tells you by
refusing (`trade increment is wrong`). The helper walks a ladder rather than
hard-coding a table that will drift.

**A round amount rarely fits.** $5,000 of ETH becomes 2.0 ETH at a $2,150 strike —
$4,300 working, $700 not. That is a **note**, never a refusal: gating on it would
hide whichever asset's step happens to fit the amount worst.

## The constraint to design around

**Quotes expire in seconds.** Each maker sets its own `validUntil`. Quote → gas →
sign → confirm has to fit inside that, so an approval card cannot carry a quote.

So the skill never signs. It files `acp options open --venue rysk`: the card approves a
premium floor (the owner's, or 95% of the quote) and lives 5 minutes, and once it is
approved bevo-server asks Rysk again, takes a quote only if it clears the floor, and signs
it as the owner's HyperEVM smart account at once (bevo-server's
`docs/options-venues.md` § Rysk). The helper's quotes are for choosing, never for acting
on.

## Why there is a WebSocket in the helper

Rysk is RFQ over a socket and the container ships no websocket package, so the helper
carries a small stdlib handshake-plus-two-frames client. Reads and quotes need no key
and no account.

## Validating a change

```sh
curl -sSLO https://virtual-protocol.github.io/butler-skills/tools/validate.py
python3 validate.py --standalone .
```

## Releasing

Bump `version` in `SKILL.md`, add the `CHANGELOG.md` entry, merge to `main`. Listing
is maintainer-only and a money-moving skill needs two maintainer reviews.

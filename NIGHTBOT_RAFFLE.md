# Nightbot Raffle Selection Algorithm — Findings

Source file: `index-DW5xu8tp.js` (minified/bundled frontend JS)

---

## Giveaway Modes

Three modes exist (`$e` enum, line 286–291):

| Value | Label |
|-------|-------|
| `active_user` | Active Chatters |
| `keyword` | Keyword Raffle |
| `random_number` | Random Number (different mechanism, not a weighted raffle) |

Weighting is only available in `active_user` and `keyword` modes (`pd = new Set([$e.ACTIVE_USER, $e.KEYWORD])`, line 624).

---

## Entry Collection

Users are added to the pool as they send chat messages:

- **Active Chatters mode**: any user who sends a message is entered automatically.
- **Keyword mode**: a user must include the configured phrase in their message to enter.
  - If an already-entered user types the keyword again, they receive a `KEYWORD_SPAM` tag.
- The channel owner is never entered.
- Each user's `activeAt` timestamp is updated on every message they send.

---

## Eligibility (re-evaluated at draw time)

When "Choose a Winner" is clicked, all entrants are re-checked. A user is **ineligible** if they carry any of the following tags:

| Tag | Condition |
|-----|-----------|
| `WRONG_USER_LEVEL` | Does not meet the required role (subscriber, regular, mod, etc.) |
| `NOT_ACTIVE` | Has not chatted within the configured activity timeout window (Active Chatters mode only; checked every 3 seconds and again at draw time) |
| `MANUAL_REMOVAL` | Manually removed by the streamer |
| `WINNER` | Has already won — only applied when **Unique Winners** is enabled |
| `KEYWORD_SPAM` | Typed the keyword more than once — only applied when **Prevent Phrase Spam** is enabled |

---

## Weighting (Ticket Pool)

If any weight differs from `1`, each eligible user is duplicated into the pool a number of times equal to their role's weight (lines 12914–12918):

```js
$ = $.flatMap((user) =>
    Array.from({ length: r[user.userType] ?? 1 }, () => user)
);
```

- `r` is the `luck` object mapping each user role to a numeric multiplier.
- If a role has no configured weight, it defaults to `1`.
- A subscriber with weight `2` therefore appears **twice** in the pool; a regular with weight `1.5` appears once... however, weights appear to be whole integers in practice given the `Array.from({ length })` call.
- The total pool size (including duplicates) is what the random index is drawn from.

---

## Random Number Generation — drand (Verifiably Fair)

Rather than `Math.random()`, Nightbot uses **[drand](https://drand.love/)** — Cloudflare's publicly verifiable, decentralized randomness beacon. This makes draws provably fair and auditable by anyone.

### Fetching randomness (`Wj`, lines 12240–12253)

```
GET https://drand.cloudflare.com/52db9ba70e0cc0f6eaf7803dd07447a1f5477735fd3f661792ba94600c84e971/public/latest
```

Returns:
```json
{ "round": <uint>, "randomness": "<64-char hex>", "signature": "<hex>" }
```

- Retries up to **10 times** (with 1-second delays) until a **new** round is received (round number must be greater than the last used round, tracked in `$r`).
- Throws if the round number doesn't advance after 10 attempts.

### Unbiased index selection (`Bj(min, max)`, lines 12254–12293)

Uses **rejection sampling** to eliminate modulo bias:

```
va    = 4294967295          // 2³² − 1 (max uint32)
range = max − min + 1       // number of pool slots
n     = va − (va % range)   // rejection limit

Parse the 64-char hex randomness string into 32 raw bytes.
Treat them as 8 consecutive uint32 values (big-endian).

For each uint32 value c:
    if c < n:
        return (c % range) + min   ← unbiased result
    else:
        skip (would introduce bias)

If all 8 values are rejected → fetch a fresh drand round and retry.
```

- Special case: if `min === max` (pool of 1), returns `min` immediately with `drandRound = 0` (no network call needed).
- Maximum supported pool size is `4294967295` entries.

The winner is `pool[randomIndex]`.

---

## Duplicate-Winner Guard

If the newly selected winner is the **same person** as the winner currently displayed on screen, the draw is retried recursively — up to **3 times** — before accepting the duplicate (lines 12921–12924).

---

## Verification

The result object stored after a draw contains (lines 12929–12934):

| Field | Description |
|-------|-------------|
| `drandRound` | The beacon round number used |
| `drandSignature` | The beacon's BLS signature for that round |
| `drandRandomness` | The raw 64-char hex randomness value |
| `chunkIndex` | Which 4-byte chunk of the randomness was used |
| `rawUint32` | The raw uint32 value that was accepted |
| `rejectionLimit` | The `n` threshold used for rejection sampling |
| `range` | The total pool size |

If `drandRound > 0`, these fields are displayed in the UI so anyone can independently verify the draw was fair by re-running the same rejection-sampling arithmetic against the public beacon data.

---

## Summary

1. Eligible users are collected into a pool; weighted users are duplicated N× (ticket pool model).
2. A provably-fair random index is drawn from Cloudflare's drand beacon.
3. Rejection sampling ensures perfectly uniform distribution with no modulo bias.
4. The full draw parameters are published so results are independently verifiable.

# Polymarket CLOB Order Types

Quick reference for the four order types available on Polymarket's CTF Exchange orderbook.

## Summary

| Type | Partial fills? | Stays open? | Use when... |
|---|---|---|---|
| **GTC** | Yes | Yes — forever | Price matters more than speed |
| **GTD** | Yes | Yes — until deadline | Price matters, but market expires |
| **FAK** | Yes | No — kills unfilled instantly | Speed matters, partial OK |
| **FOK** | No | No — all or nothing | Size must be exact |

## Details

### GTC — Good 'Til Cancelled
Limit order that sits on the book until filled or cancelled. Risk: can go unfilled forever if price runs away from your limit. Best for passive strategies where price matters more than speed.

### FAK — Fill-And-Kill
Take whatever is available NOW at best price, kill the rest. Partial fills allowed — whatever doesn't fill immediately is cancelled. Best for needing to get in or out right now, can't risk an open order hanging.

### FOK — Fill-Or-Kill
Fill ALL shares immediately, or cancel the entire order. Difference from FAK: FAK gives partial fills, FOK gives all or nothing. Best for size-sensitive strategies where partial fill changes P/L math.

### GTD — Good 'Til Date
Like GTC but with an expiration timestamp. Auto-cancels at the deadline. Best for markets that resolve at a known time (BTC 5-min markets!).

## What we use

| Strategy | Type | Why |
|---|---|---|
| Momentum follower | FAK | Must enter instantly at T+45-85s. Partial fill better than no fill |
| Midpoint scalp (when running) | FAK market sell | Must exit at TP/spike. GTC limit sits unfilled; FOK killed on thin liquidity |
| Directional | Paper only | Would use FAK if live — entry window is 60-180s |

## Pitfall: FOK market sells get killed silently

Polymarket requires 5-share minimum. Markets with thin liquidity (only 2-3 shares available) will KILL a 5-share FOK entirely — you get zero. Switched to FAK market sells for all exit paths (TP, spike, near-close) to get partial fills.

## Pitfall: OrderType.IOC does NOT exist

`py-clob-client-v2` has `FOK`, `FAK`, `GTC`, `GTD` — no `IOC`. Using it causes `AttributeError` on every sell attempt.

# Polymarket CLOB Order Types

## FAK — Fill-And-Kill

Take whatever is available NOW at the best price, cancel the rest immediately. Partial fills allowed.

```
You want 5 shares. Orderbook has 3 at $0.72.
→ You get 3 shares at $0.72. Remaining 2 cancelled immediately.
```

- ✅ Guarantees immediate execution or nothing
- ✅ No orphan orders left on the book
- ✅ Best for: momentum entry, directional entry, closing positions fast
- ❌ Might not fill full size if liquidity is thin

**This is what we use for momentum follower live buys.** It replaced GTC because GTC left phantom unfilled orders that the DB assumed filled.

## GTC — Good 'Til Cancelled

Places a limit order that sits on the book until filled or cancelled.

```
Buy 5 shares at $0.70. Order sits for hours/days until someone sells at $0.70.
```

- ✅ Gets your exact price (if it fills)
- ❌ Can go unfilled forever
- ❌ Creates phantom trades — DB assumes filled, exchange says unfilled
- Best for: passive strategies where price matters more than speed

## FOK — Fill-Or-Kill

Fill ALL shares immediately, or cancel the ENTIRE order. No partial fills.

```
You want 5 shares. Orderbook has 3. FOK cancels everything — you get ZERO.
```

- ✅ All or nothing — no partial fills
- ❌ More likely to fail if liquidity is thin
- Best for: size-sensitive strategies where a partial fill changes P/L math

## GTD — Good 'Til Date

Like GTC but with an expiration timestamp.

```
Buy at $0.70, auto-cancel at 5:00 PM if unfilled.
```

- Best for: 5-min BTC markets where you can't leave an order past resolution

## Comparison

| Type | Partial fills? | Stays open? | When to use |
|---|---|---|---|
| GTC | Yes | Forever | Price > speed |
| GTD | Yes | Until deadline | Market expires |
| FAK | Yes | No | Speed > price |
| FOK | No | No | Exact size required |

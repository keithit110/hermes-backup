# Polymarket CLOB Market Sell Order Types

## Available OrderType values (py-clob-client-v2)

```python
from py_clob_client_v2 import OrderType
# Available: FOK, FAK, GTC, GTD
# NOT available: IOC (does not exist — causes AttributeError)
```

## Pitfall: `OrderType.IOC` doesn't exist (2026-06-29)

`py-clob-client-v2` has **no** `OrderType.IOC`. Using it causes:

```
AttributeError: type object 'OrderType' has no attribute 'IOC'
```

The equivalent is **FAK** (Fill-and-Kill) — fills whatever shares are available immediately, cancels any unfilled portion. Same semantics as IOC.

### Order type cheat sheet

| Type | Behavior | Use case |
|------|----------|----------|
| **FOK** | Fill entire order or kill it entirely | Not recommended — gets killed when liquidity is insufficient for all shares |
| **FAK** | Fill what's available, cancel rest | **Market sell** — get out now at best available price |
| **GTC** | Stay on book until filled or cancelled | Limit orders (buys, TP sells) |
| **GTD** | Good-til-date — expires at a specific time | Rarely needed for 5-min windows |

### Correct FAK market sell implementation

```python
from py_clob_client_v2 import MarketOrderArgs, OrderType, Side

order_args = MarketOrderArgs(
    token_id=token_id,
    amount=size,        # number of shares for SELL, USDC for BUY
    side=Side.SELL,
)
result = _client.create_and_post_market_order(
    order_args=order_args,
    order_type=OrderType.FAK,  # NOT IOC — doesn't exist!
)
```

### FOK failure pattern

When using FOK and the orderbook lacks enough depth for all shares:

```
PolyApiException: "order couldn't be fully filled. FOK orders are fully filled or killed."
```

The entire order is killed — zero shares sold. This is why limit sells at bid were attempted first, but FAK market orders are the correct fix for true market sells.

### Why limit sells at bid were also attempted (and why they're NOT the right approach)

GTC limit sells at the current bid work but place a resting order — if the market moves against you before the order fills, you're stuck with an unfilled limit. FAK market orders immediately fill whatever's available at the market price and cancel the rest. For take-profit exits, FAK is the correct choice.

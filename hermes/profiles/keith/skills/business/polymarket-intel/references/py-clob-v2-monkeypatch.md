# py-clob-client-v2 POLY_1271 L1 Auth Monkeypatch

## Problem

The `py-clob-client-v2` SDK (v1.0.1) has a bug in the L1 authentication path:
- `create_level_1_headers()` always uses `signer.address()` (the EOA) as `POLY_ADDRESS`
- `sign_clob_auth_message()` always bakes the EOA into the ClobAuth EIP-712 message
- `ClobClient._l1_headers()` doesn't pass `signature_type` or `funder` to `create_level_1_headers()`

This means even with `signature_type=POLY_1271` and `funder=deposit_wallet`, the API key is ALWAYS bound to the EOA. Orders then fail with:
```
HTTP 400: {"error": "the order signer address has to be the address of the API KEY"}
```

Ref: https://github.com/Polymarket/py-clob-client-v2/issues/70

## Fix

Insert this monkeypatch in `live_trading.py`'s `init()` function AFTER the SDK imports but BEFORE any `ClobClient` instantiation.

```python
from py_clob_client_v2 import ApiCreds, ClobClient, SignatureTypeV2

# ── MONKEYPATCH: fix SDK bug where L1 auth always binds API key to EOA ──
from py_clob_client_v2.headers import headers as _hdr_mod
from py_clob_client_v2.signing import eip712 as _eip712_mod

def _patched_sign_clob_auth_message(signer, timestamp, nonce, funder_address=None):
    """Sign ClobAuth with optional deposit wallet override."""
    actual_address = funder_address if funder_address else signer.address()
    from eth_utils import keccak
    from py_order_utils.utils import prepend_zx
    from py_clob_client_v2.signing.model import ClobAuth
    from py_clob_client_v2.signing.eip712 import get_clob_auth_domain, MSG_TO_SIGN
    clob_auth_msg = ClobAuth(
        address=actual_address,
        timestamp=str(timestamp),
        nonce=nonce,
        message=MSG_TO_SIGN,
    )
    chain_id = signer.get_chain_id()
    auth_struct_hash = prepend_zx(
        keccak(clob_auth_msg.signable_bytes(get_clob_auth_domain(chain_id))).hex()
    )
    return prepend_zx(signer.sign(auth_struct_hash))

def _patched_create_l1(signer, nonce=None, timestamp=None,
                       signature_type=None, funder=None):
    """Create L1 headers — uses funder address for POLY_1271."""
    from datetime import datetime
    ts = timestamp if timestamp is not None else int(datetime.now().timestamp())
    n = nonce if nonce is not None else 0
    use_funder = (
        signature_type is not None
        and int(signature_type) == 3  # POLY_1271
        and funder
    )
    funder_addr = funder if use_funder else None
    signature = _patched_sign_clob_auth_message(signer, ts, n, funder_address=funder_addr)
    poly_addr = funder if use_funder else signer.address()
    return {
        _hdr_mod.POLY_ADDRESS: poly_addr,
        _hdr_mod.POLY_SIGNATURE: signature,
        _hdr_mod.POLY_TIMESTAMP: str(ts),
        _hdr_mod.POLY_NONCE: str(n),
    }

_hdr_mod.create_level_1_headers = _patched_create_l1

# Patch ClobClient._l1_headers to pass sig_type + funder
def _patched_l1_headers(self, nonce=None):
    self.assert_level_1_auth()
    return _patched_create_l1(
        self.signer,
        nonce=nonce,
        timestamp=self._get_timestamp(),
        signature_type=getattr(self.builder, 'signature_type', None),
        funder=getattr(self.builder, 'funder', None),
    )

ClobClient._l1_headers = _patched_l1_headers
# ── END MONKEYPATCH ──
```

## Limitations

This monkeypatch is **necessary but not sufficient** for order placement. The deposit wallet proxy must ALSO be fully deployed on-chain (~250+ bytes of bytecode). Without the proxy deployed, the backend's EIP-1271 signature verification fails regardless of what `POLY_ADDRESS` is sent.

## Verified state (2026-06-28)

- py-clob-client-v2 version: 1.0.1
- Installed at: `/usr/local/lib/python3.11/site-packages/py_clob_client_v2/`
- Bug confirmed in headers/headers.py and signing/eip712.py
- Monkeypatch applied in `/root/polymarket-intel/app/live_trading.py`
- After monkeypatch: `create_api_key()` succeeds when proxy is deployed
- Without proxy deployment: orders still fail with signer error

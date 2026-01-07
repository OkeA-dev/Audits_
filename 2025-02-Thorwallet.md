# [MEDIUM-01] Incorrectly Flagging Users as Bridged Token Holders

**Description and Impact:**
The `_credit` function in the TITN contract is responsible for crediting tokens when received via LayerZero’s Omnichain Fungible Token (OFT) mechanism. However, this function incorrectly flags all credited addresses as bridged token holders, regardless of whether the transaction originated from another chain or occurred locally on BASE.

Key Feature Broken:
According to the protocol’s design:

Non-bridged TITN Tokens on BASE should be freely transferable.
Bridged TITN Tokens should be restricted until explicitly unlocked.
However, this issue violates the intended behavior by incorrectly restricting local TITN holders on BASE, preventing them from freely transferring their tokens.

**Recommended Mitigation:**
Modify _credit to ensure that bridging restrictions only apply to tokens that actually came from another chain.
```diff
function _credit(
    address _to,
    uint256 _amountLD,
    uint32 _srcEid
) internal virtual override returns (uint256 amountReceivedLD) { 
    if (_to == address(0x0)) _to = address(0xdead);
    _mint(_to, _amountLD);

+    uint32 baseChainId = 8453; // BASE Mainnet Chain ID
    // Only flag as a bridged token holder if tokens came from another chain
+    if (_srcEid != baseChainId && !isBridgedTokenHolder[_to]) { 
        isBridgedTokenHolder[_to] = true;
    }

    return _amountLD;
}
```
# [HIGH-01] Incorrect Token Transfer Destination in _forceRepay Function
**Summary:** The `_forceRepay` function incorrectly transfers yield tokens to the contract itself instead of the transmuter.
**Description:** In the `_forceRepay` function, yield tokens are transferred to `address(this)` instead of the `transmuter`:
```solidity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // ...
    // Transfer the repaid tokens from the account to the transmuter.
    TokenUtils.safeTransfer(yieldToken, address(this), creditToYield); //@audit-issue incorrect address transferring to. address(this) instead of the transmuter.
    return creditToYield;
}
```
This means that tokens intended to go to the transmuter are instead stuck in the contract, preventing proper protocol functioning.

**Impact:**
This bug prevents the transmuter from receiving repaid yield tokens, which breaks core protocol functionality. Any debt that should be repaid through forced repayments during liquidations will not actually be processed by the transmuter.
**Recommended Mitigation:**
Fix the token transfer destination in the `_forceRepay` function:
```soldity
function _forceRepay(uint256 accountId, uint256 amount) internal returns (uint256) {
    // ...
    // Transfer the repaid tokens from the account to the transmuter.
    TokenUtils.safeTransfer(yieldToken, transmuter, creditToYield);
    return creditToYield;
}
```

# [Informational-01] Missing Validation for Alchemist Transmuter Migration
**Summary:**
The `createRedemption` function lacks validation to verify that the transmuter is still active within the alchemist, potentially causing user funds to be locked if the alchemist migrates to a new transmuter.
**Description:**
When users create a redemption in the transmuter, they deposit synthetic tokens to be redeemed for yield tokens. However, there's no check to verify that the transmuter is still the active transmuter for the alchemist. If the alchemist migrates to a new transmuter, users who deposit into the old one will have their funds locked, as the old transmuter will no longer be able to redeem tokens from the alchemist.
```solidity
/// @inheritdoc ITransmuter
function createRedemption(uint256 syntheticDepositAmount) external {
    //@audit-issue: missing check to pause redemption when, alchemist had change migrated transmuter //
    //missing check if alchemist still uses the same transmuter before redemption.
    if (syntheticDepositAmount == 0) {
        revert DepositZeroAmount();
    }

    if (totalLocked + syntheticDepositAmount > depositCap) {
        revert DepositCapReached();
    }

    if (totalLocked + syntheticDepositAmount > alchemist.totalSyntheticsIssued()) { 
        revert DepositCapReached();
    }

    TokenUtils.safeTransferFrom(syntheticToken, msg.sender, address(this), syntheticDepositAmount);

    _positions[++_nonce] = StakingPosition(syntheticDepositAmount, block.number, block.number + timeToTransmute);

    // Update Fenwick Tree
    _updateStakingGraph(syntheticDepositAmount.toInt256() * BLOCK_SCALING_FACTOR / timeToTransmute.toInt256(), timeToTransmute);

    totalLocked += syntheticDepositAmount;

    _mint(msg.sender, _nonce);

    emit PositionCreated(msg.sender, syntheticDepositAmount, _nonce);
}

```
**Impact:**
Users' funds could be permanently locked in the transmuter if the alchemist migrates to a new transmuter, resulting in complete loss of deposited assets.

**Recommended Mitigation:**
Protocol specific
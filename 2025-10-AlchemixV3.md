# [CRITICAL-01] Same-block earmark early-exit leaves stale transmuter balance, causing under-earmarking

 **Summary:**
 Same-block `earmark()` early-exit leaves a stale transmuter balance, inflating “cover” next block and under‑earmarking debt, letting borrowers overborrow, delaying redemptions, and raising bad‑debt/insolvency risk.

 **Description:**
 In `AlchemistV3.sol::_earmark()`, an early-return guard `if (block.number <= lastEarmarkBlock) return;` exits on subsequent calls within the same block. When the transmuter’s MYT balance changes between two same-block calls (e.g., via repay, direct MYT transfer, or equivalent inflow), the second `_earmark()` doesn’t update `lastTransmuterTokenBalance`. On the next block, `_earmark()` computes “cover” as `transmuterCurrentBalance - lastTransmuterTokenBalance` using this stale value, which inflates the perceived cover and reduces the amount of debt earmarked below what it should be. The behavior can be triggered by two `_earmark()`-invoking actions in the same block (e.g., `poke()` twice by different users, `withdraw()+poke()`, etc.), and it’s not fully prevented by the mint/repay same-block restriction.

Vulnerable in order of Operation:
```solidity
function _earmark() internal {
    if (totalDebt == 0) return;
    if (block.number <= lastEarmarkBlock) return; // @<-- Early-exit on same block (stale balance risk)

    // Yield the transmuter accumulated since last earmark (cover)
    uint256 transmuterCurrentBalance = TokenUtils.safeBalanceOf(myt, address(transmuter));
    uint256 transmuterDifference =
        transmuterCurrentBalance > lastTransmuterTokenBalance
            ? transmuterCurrentBalance - lastTransmuterTokenBalance
            : 0;
    uint256 amount = ITransmuter(transmuter).queryGraph(lastEarmarkBlock + 1, block.number);

    uint256 coverInDebt = convertYieldTokensToDebt(transmuterDifference);
    amount = amount > coverInDebt ? amount - coverInDebt : 0;

    lastTransmuterTokenBalance = transmuterCurrentBalance; // @<-- Only updated when not early-exiting

    uint256 liveUnearmarked = totalDebt - cumulativeEarmarked;
    if (amount > liveUnearmarked) amount = liveUnearmarked;

    if (amount > 0 && liveUnearmarked != 0) {
        uint256 previousSurvival = PositionDecay.SurvivalFromWeight(_earmarkWeight);
        if (previousSurvival == 0) previousSurvival = ONE_Q128;

        uint256 earmarkedFraction = _divQ128(amount, liveUnearmarked);

        _survivalAccumulator += _mulQ128(previousSurvival, earmarkedFraction);
        _earmarkWeight += PositionDecay.WeightIncrement(amount, liveUnearmarked);

        cumulativeEarmarked += amount;
    }

    lastEarmarkBlock = block.number;
}
```
**Impact:**
The stale `lastTransmuterTokenBalance` causes the next `_earmark()` to over-count cover and under-earmark system debt, allowing borrowers to retain more unearmarked debt and therefore more borrowing headroom than intended, distorting redemption pressure and global accounting in favor of borrowers and against redeemers.

**Recommendation:**
Protocol specific.

# [Insight-01] Incorrect Taylor Series Approximation in _approxAPY() Function Drastically Understates Yield
**Summary:**
The `_approxAPY()` function in MYTStrategy.sol incorrectly divides the second-order Taylor expansion term by `SECONDS_PER_YEAR` instead of `2`, causing the Annual Percentage Yield (APY) calculation to understate compounding effects by up to 33% at realistic yield rates.
**Description:**
The `_approxAPY()` function attempts to approximate APY using a second-order Taylor series expansion for continuous compounding. The correct mathematical formula for converting a per-second rate to APY is:

`APY = e^r - 1 ≈ r + (r^2)/2 + (r^3)/6 + ...`

For a second-order approximation (which the code attempts to implement):

`APY ≈ r + (r^2)/2`

Where `r` is the annual rate calculated as: `r = ratePerSecond × SECONDS_PER_YEAR`

However, the current implement in MYTStrategy is incorrect,
```solidity
function _approxAPY(uint256 ratePerSecWad) internal pure returns (uint256) {
    uint256 apr = ratePerSecWad * SECONDS_PER_YEAR;
    uint256 aprSq = apr * apr / FIXED_POINT_SCALAR;
    return apr + aprSq / (2 * SECONDS_PER_YEAR);  //@audit-issue incorrect implementation
}
```

The current implementation computes: WRONG: `APY = r + (r^2) / (2 × SECONDS_PER_YEAR)`

Where:

`r = ratePerSec × SECONDS_PER_YEAR (annual rate)`

The second term becomes: `(ratePerSec × SECONDS_PER_YEAR)^2 / (2 × SECONDS_PER_YEAR)`

Simplifying: `(ratePerSec^2 × SECONDS_PER_YEAR) / 2`

This is NOT the correct Taylor expansion. The correct second-order term should be: `(r^2) / 2 = (ratePerSec × SECONDS_PER_YEAR)^2 / 2`

**Root Cause:**
The bug divides the quadratic compounding term by `SECONDS_PER_YEAR (31,536,000)`, making the compounding effect approximately 31 million times smaller than it should be. This causes the function to essentially return APR instead of APY, completely negating the purpose of the compound yield calculation.
```solidity
return apr + aprSq / (2 * SECONDS_PER_YEAR);  // Line 244 ->>
```

**Recommended Mitigation:**
Replace the incorrect division by `(2 * SECONDS_PER_YEAR)` with division by `2`:

```diff
function _approxAPY(uint256 ratePerSecWad) internal pure returns (uint256) {
    uint256 apr = ratePerSecWad * SECONDS_PER_YEAR;
    uint256 aprSq = apr * apr / FIXED_POINT_SCALAR;
-     return apr + aprSq / (2 * SECONDS_PER_YEAR);
+     return apr + aprSq / 2;
}
```

# [Insight-02] Receipt Token Misconfiguration in Aave Strategies
**Summary:**
Aave strategies approve wrong token to Permit2, blocking emergency migration mechanism during protocol upgrades

**Description:**
The Aave V3 strategy implementations pass the wrong token (underlying asset) instead of the receipt token (aToken) to the parent `MYTStrategy` constructor. This causes `Permit2` to receive approval for a token the strategy doesn't hold, while having no approval for the token the strategy actually holds.

**Root Cause:**
In all three Aave strategies, the constructor passes the underlying asset (_usdc or _weth) instead of the Aave receipt token (_aUSDC or _aWETH) as the fourth parameter to MYTStrategy:
```solidity
// WRONG - Current Implementation
MYTStrategy(_myt, _params, _permit2Address, _usdc)

// CORRECT - Should be
MYTStrategy(_myt, _params, _permit2Address, _aUSDC)
```
The MYTStrategy constructor then approves this token to Permit2:
```solidity
// MYTStrategy.sol:106
IERC20(receiptToken).approve(permit2Address, type(uint256).max);
```

**Impact:**
Defeats the purpose of Permit2 integration for these strategies



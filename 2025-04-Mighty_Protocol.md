# [HIGH-01] Unadjusted Pyth Oracle Prices Cause Misaligned Token Valuations and Inaccurate Position Calculations
**Summary**
The protocol incorrectly uses raw price values returned by the Pyth oracle without aligning them with the corresponding token decimals. Since Pyth expresses prices using a base value and an exponent, and each token has its decimal configuration, failing to normalise these values leads to inaccurate asset pricing throughout the protocol. This misalignment affects all token valuation calculations, including position and debt tracking in the `ShadowRangeVault`.
**Description:**
In the current implementation, the protocol retrieves price data from the Pyth oracle but directly uses the raw price field without applying the `expo` (exponent) or adjusting it to match the token's decimals. This results in mismatches between the price precision and token units.
```solidity
function getPythPrice(address token) public view returns (uint256) {
    bytes32 priceId = pythPriceIds[token];
    if (priceId == bytes32(0)) {
        revert("PriceId not set");
    }
    PythStructs.Price memory priceStruct = pyth.getPriceUnsafe(priceId);
   
    require(priceStruct.publishTime + maxPriceAge > block.timestamp, "Price is too old");
    uint256 price = uint256(uint64(priceStruct.price));
    return price;
}
```
**Technical Background**
Pyth represents price as a signed integer (price) along with a signed exponent (expo), meaning the actual price is computed as:

actual_price = price * 10^expo
This implementation ignores the exponent entirely and does not scale the resulting value to match the token's decimal standard (e.g., 6, 8, or 18 decimals). Consequently, the price used in calculations is orders of magnitude off from its true value.

**Affected Calculations in ShadowRangeVault**
The incorrectly scaled prices are propagated into core vault operations:
```solidity
function getPositionValue(uint256 positionId) public view returns (uint256 value) {
    (uint256 amount0, uint256 amount1) = getPositionAmounts(positionId);
    return amount0 * getTokenPrice(token0) / 10 ** token0Decimals
        + amount1 * getTokenPrice(token1) / 10 ** token1Decimals;
}

function getDebtValue(uint256 positionId) public view returns (uint256 value) {
    (uint256 amount0Debt, uint256 amount1Debt) = getPositionDebt(positionId);
    return amount0Debt * getTokenPrice(token0) / 10 ** token0Decimals
        + amount1Debt * getTokenPrice(token1) / 10 ** token1Decimals;
}
```
Since getTokenPrice ultimately returns the unadjusted raw price from Pyth, the downstream calculations operate on incorrect values that do not align with the real token prices or units.

**Impact:**
Using unadjusted Pyth prices that don’t match token decimal formats results in significant valuation errors. This can lead to:

* Incorrect position value and debt assessments.

* Mispricing of collateral and liabilities.

* Unfair liquidations or protocol insolvency risks.

* Inaccurate health metrics for vaults and users.

**Recommended Mitigation**
Modify the getPythPrice function to correctly apply the Pyth exponent and normalize the result to match a consistent decimal format (e.g., 8 decimals). Here is the corrected implementation:
```solidity
function getPythPrice(address token) public view returns (uint256) {
    bytes32 priceId = pythPriceIds[token];
    if (priceId == bytes32(0)) {
        revert("PriceId not set");
    }
    PythStructs.Price memory priceStruct = pyth.getPriceUnsafe(priceId);
   
    require(priceStruct.publishTime + maxPriceAge > block.timestamp, "Price is too old");
    
    // Extract price and exponent
    int64 price = priceStruct.price;
    int32 expo = priceStruct.expo;
    
    // Calculate adjusted price based on exponent
    uint256 adjustedPrice;
    if (expo < 0) {
        // For negative exponents (division)
        adjustedPrice = uint256(price) * (10 ** uint256(8)) / (10 ** uint256(-expo));
    } else {
        // For positive exponents (multiplication)
        adjustedPrice = uint256(price) * (10 ** uint256(expo + 8));
    }
    
    // Return price scaled to 8 decimals for consistency
    return adjustedPrice;
}
```
By normalising to 8 decimals, this approach ensures pricing precision is consistent and aligns with token units used across the protocol, thereby maintaining calculation integrity.

# [LOW-01] Liquidation Not Triggered at Threshold Due to Strict Inequality Check
**Summary**
The liquidation eligibility check uses a strict inequality (`>`) instead of a non-strict one (`>=`). This prevents liquidation when a position's debt ratio is exactly equal to the predefined `liquidationDebtRatio`, allowing risky positions to remain active longer than intended.
**Description:**
The `getDebtRatio` function calculates the current debt ratio of a position:
```solidity
function getDebtRatio(uint256 positionId) public view returns (uint256 debtRatio) {
    return getDebtValue(positionId) * 10000 / getPositionValue(positionId);
}
```
In the liquidation logic, the eligibility is checked with:
```solidity
require(getDebtRatio(positionId) > liquidationDebtRatio, "Debt ratio is too low");
```
This condition only triggers liquidation if the current debt ratio is strictly greater than the threshold. If the debt ratio is exactly equal to liquidationDebtRatio, the position is still considered safe, even though it's at maximum acceptable risk.

This opens a small but critical window where high-risk positions avoid liquidation due to a logic flaw.

**Impact:**
High, because a borrower can easily monitor and maintain their debt ratio just below or exactly at the threshold to avoid liquidation.

**Recommended Mitigation**
Change the liquidation condition to use a non-strict inequality:
```solidity
require(getDebtRatio(positionId) >= liquidationDebtRatio, "Debt ratio is too low");
```
This ensures that positions at or above the threshold are treated consistently and fairly, and are eligible for liquidation as intended.
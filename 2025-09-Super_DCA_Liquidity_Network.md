# [HIGH-01] Incompatible with Native ETH Pools Due to ERC20 Assumptions
**Summary:**
The `collectFees` function assumes all pool currencies are ERC20 tokens, making it incompatible with Uniswap V4 pools that contain native ETH. This prevents fee collection from ETH/token pairs and could lead to failed transactions or lost fees.
**Description:**
Uniswap V4 introduces native ETH support where Currency can represent either ERC20 tokens or native ETH (address(0)). The `collectFees` function incorrectly assumes all currencies are ERC20 tokens by using `IERC20(Currency.unwrap(token)).balanceOf()`, which will fail for native ETH since address(0) is not a valid ERC20 contract.
**Root Cause:**
```solidity
function collectFees(uint256 nfpId, address recipient) external override {  
    _checkOwner();  
  
    // Validate the NFT ID and recipient address  
    if (nfpId == 0) revert SuperDCAListing__UniswapTokenNotSet();  
    if (recipient == address(0)) revert SuperDCAListing__InvalidAddress();  
  
    // Retrieve the position's pool information  
    (PoolKey memory key,) = POSITION_MANAGER_V4.getPoolAndPositionInfo(nfpId);  
    Currency token0 = key.currency0;  
    Currency token1 = key.currency1;  
  
    // Record token balances before fee collection  
    uint256 balance0Before = IERC20(Currency.unwrap(token0)).balanceOf(recipient); // @audit-issue2: Fails for native ETH  
    uint256 balance1Before = IERC20(Currency.unwrap(token1)).balanceOf(recipient); // @audit-issue2: Fails for native ETH  
```
The issue extends to the entire fee collection logic, as Uniswap V4's TAKE_PAIR action with native ETH requires special handling that the current implementation doesn't provide.

**Impact:**
Contract cannot collect fees from pools containing native ETH, limiting protocol compatibility with major trading pairs and potentially locking fee revenue.

**Recommended Mitigation:**
Implement proper native ETH handling using Uniswap V4's currency library and balance accounting:
```diff

```import {CurrencyLibrary} from "@uniswap/v4-core/src/types/Currency.sol";  
  
function collectFees(uint256 nfpId, address recipient) external override {  
    _checkOwner();  
    if (nfpId == 0) revert SuperDCAListing__UniswapTokenNotSet();  
    if (recipient == address(0)) revert SuperDCAListing__InvalidAddress();  
  
    (PoolKey memory key,) = POSITION_MANAGER_V4.getPoolAndPositionInfo(nfpId);  
    Currency token0 = key.currency0;  
    Currency token1 = key.currency1;  
  
-  uint256 balance0Before = IERC20(Currency.unwrap(token0)).balanceOf(recipient);  
-  uint256 balance1Before = IERC20(Currency.unwrap(token1)).balanceOf(recipient);  
  
    // Use CurrencyLibrary for native ETH compatible balance checking  
+ uint256 balance0Before = CurrencyLibrary.balanceOf(token0, recipient);  
+ uint256 balance1Before = CurrencyLibrary.balanceOf(token1, recipient);  
  
    // Prepare actions for fee collection  
    bytes memory actions = abi.encodePacked(uint8(Actions.DECREASE_LIQUIDITY), uint8(Actions.TAKE_PAIR));  
      
    bytes[] memory params = new bytes[](2);  
    params[0] = abi.encode(nfpId, uint256(0), uint128(0), uint128(0), bytes(""));  
    params[1] = abi.encode(token0, token1, recipient);  
  
    uint256 deadline = block.timestamp + 60;  
    POSITION_MANAGER_V4.modifyLiquidities(abi.encode(actions, params), deadline);  
  
- uint256 balance0After = IERC20(Currency.unwrap(token0)).balanceOf(recipient);  
- uint256 balance1After = IERC20(Currency.unwrap(token1)).balanceOf(recipient);  
  
    // Use CurrencyLibrary for post-collection balance checks  
+ uint256 balance0After = CurrencyLibrary.balanceOf(token0, recipient);  
+ uint256 balance1After = CurrencyLibrary.balanceOf(token1, recipient);  
  
    uint256 collectedAmount0 = balance0After - balance0Before;  
    uint256 collectedAmount1 = balance1After - balance1Before;  
  
    emit FeesCollected(  
        recipient, Currency.unwrap(token0), Currency.unwrap(token1), collectedAmount0, collectedAmount1  
    );  
}  
```

# [HIGH-02] Reward Loss Vulnerability via Index Reset During Stake/Unstake
**Summary:**
The `accrueReward` function suffers from a reward calculation flaw where staking or unstaking resets the token's reward index, causing loss of accrued rewards for all stakers in that token bucket when rewards are claimed immediately after position changes.

**Description:**
When users stake or unstake, the token's `lastRewardIndex` is updated to the current global rewardIndex. If `accrueReward` is called shortly after, the delta calculation `(rewardIndex - info.lastRewardIndex)` becomes near-zero, resulting in minimal or zero rewards despite significant time elapsed since the last actual reward accrual. This creates a window where rewards that should be distributed are effectively lost.

**Attack Path Trace**
Step 1: Normal Reward Accumulation
```bash
Time T0: lastRewardIndex = 100, rewardIndex = 100  
Time T1: rewardIndex grows to 150 (50 units of rewards accumulated)  
accrueReward() called → delta = 150 - 100 = 50 → rewards distributed ✓  
```
Step 2: Attack Vector - Stake/Unstake Reset
```bash
Time T0: lastRewardIndex = 100, rewardIndex = 100  
Time T1: User stakes → lastRewardIndex updated to current rewardIndex (100)  
Time T2: accrueReward() called → delta = 100 - 100 = 0 → NO rewards distributed ✗  
Time T3: rewardIndex grows to 150 (50 units of rewards permanently lost)  
```
Step 3: Code Execution Path
```solidity
// In stake() function:  
function stake(address token, uint256 amount) external override {  
    _updateRewardIndex(); // Updates global rewardIndex  
    // ...  
    TokenRewardInfo storage info = tokenRewardInfoOf[token];  
    info.lastRewardIndex = rewardIndex; // @audit-issue1: Resets token's index to current global index  
    // ...  
}  
  
// In unstake() function:    
function unstake(address token, uint256 amount) external override {  
    _updateRewardIndex(); // Updates global rewardIndex  
    // ...  
    TokenRewardInfo storage info = tokenRewardInfoOf[token];  
    info.lastRewardIndex = rewardIndex; // @audit-issue1: Same reset issue  
    // ...  
}  
  
// In accrueReward() function:  
function accrueReward(address token) external override onlyGauge returns (uint256 rewardAmount) {  
    _updateRewardIndex(); // Updates global rewardIndex  
      
    TokenRewardInfo storage info = tokenRewardInfoOf[token];  
    if (info.stakedAmount == 0) return 0;  
  
    uint256 delta = rewardIndex - info.lastRewardIndex; // @audit-issue1: If stake/unstake just happened, delta ≈ 0  
    if (delta == 0) return 0;  
  
    rewardAmount = Math.mulDiv(info.stakedAmount, delta, 1e18);  
    info.lastRewardIndex = rewardIndex; // Properly resets after reward calculation  
    return rewardAmount;  
}  
```
**Impact:**
Rewards accumulated between the last `accrueReward` call and any stake/unstake operation are permanently lost, unfairly penalizing stakers and reducing the protocol's reward distribution efficiency.

**Recommended Mitigation:**
Remove the premature index reset from stake/unstake functions and only update lastRewardIndex after reward calculation:
```diff
function stake(address token, uint256 amount) external override {  
    if (amount == 0) revert SuperDCAStaking__ZeroAmount();  
    if (gauge == address(0)) revert SuperDCAStaking__ZeroAddress();  
    if (!ISuperDCAGauge(gauge).isTokenListed(token)) revert SuperDCAStaking__TokenNotListed();  
  
    _updateRewardIndex();  
  
    IERC20(DCA_TOKEN).transferFrom(msg.sender, address(this), amount);  
  
    TokenRewardInfo storage info = tokenRewardInfoOf[token];  
      
    // Remove: info.lastRewardIndex = rewardIndex; //  << Remove this line  
      
    info.stakedAmount += amount;  
    totalStakedAmount += amount;  
    userStakes[msg.sender][token] += amount;  
    userTokenSet[msg.sender].add(token);  
  
    emit Staked(token, msg.sender, amount);  
}  
  
function unstake(address token, uint256 amount) external override {  
    if (amount == 0) revert SuperDCAStaking__ZeroAmount();  
  
    TokenRewardInfo storage info = tokenRewardInfoOf[token];  
    if (info.stakedAmount < amount) revert SuperDCAStaking__InsufficientBalance();  
    if (userStakes[msg.sender][token] < amount) revert SuperDCAStaking__InsufficientBalance();  
  
    _updateRewardIndex();  
  
    // Remove: info.lastRewardIndex = rewardIndex; // << Remove this line  
      
    info.stakedAmount -= amount;  
    totalStakedAmount -= amount;  
    userStakes[msg.sender][token] -= amount;  
  
    if (userStakes[msg.sender][token] == 0) {  
        userTokenSet[msg.sender].remove(token);  
    }  
  
    IERC20(DCA_TOKEN).transfer(msg.sender, amount);  
    emit Unstaked(token, msg.sender, amount);  
}  
```
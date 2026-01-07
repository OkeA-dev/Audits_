# [MEDIUM-01] JackpotBridgeManager overcharges cross-chain users when ticket price is updated mid-drawing
**Summary:** The `JackpotBridgeManager.buyTickets()` function incorrectly uses the global `ticketPrice` instead of the current drawing's ticket price when charging users. This causes cross-chain users to be overcharged when the owner updates the ticket price during an active drawing, with excess funds becoming stuck in the bridge manager contract.
**Description:** 
In `JackpotBridgeManager.sol` at line 178, the `buyTickets()` function retrieves the ticket price using:
```solidity
uint256 ticketPrice = jackpot.ticketPrice();
```
This fetches the global `ticketPrice` variable from the Jackpot contract, which represents the pending price intended for the next drawing after settlement. However, the correct price for the current drawing is stored in `drawingState[currentDrawingId]`.`ticketPrice`, which is immutable once the drawing begins.

The Jackpot contract's price architecture works as follows:

* The global `ticketPrice` variable is the "pending" price that will be copied to the next drawing during `_setNewDrawingState()`
* Each drawing's `DrawingState.ticketPrice` is set once at creation and never changes
* When `setTicketPrice()` is called, it updates the global variable but does not affect the current active drawing
  
This creates a price mismatch in the bridge manager:

1. Line 181: Bridge manager transfers `ticketPrice * tickets.length` from user (using global price)
2. Line 183: Bridge manager approves the same amount to Jackpot
3. Line 184: Jackpot's `buyTickets()` charges bridge manager using `currentDrawingState.ticketPrice` (drawing price)
When the global price is higher than the current drawing's price, the bridge manager collects more USDC from users than it needs to pay the Jackpot contract, and the excess becomes permanently stuck in the bridge manager contract.


**Impact:** High Severity:

* Direct fund loss: Cross-chain users are systematically overcharged when ticket price increases
* Funds stuck: Excess USDC accumulates in bridge manager with no withdrawal mechanism
* DoS condition: Cross-chain ticket purchases completely fail when ticket price decreases
* User trust violation: Cross-chain users pay significantly more than direct buyers for identical tickets
* No recovery: Lost funds are permanently inaccessible to users

**Recommended Mitigation:**
Modify `JackpotBridgeManager.buyTickets()` to fetch the ticket price from the current drawing state instead of the global variable:
```diff
function buyTickets(
    Ticket[] memory tickets,
    address receipient,
    address[] calldata referrers,
    uint256[] calldata referrerSharesOfWinnings,
    bytes32 nonce
) external {
    uint256 currentDrawingId = jackpot.currentDrawingId();
-   uint256 ticketPrice = jackpot.ticketPrice();
+   // Get the immutable price for the current drawing, not the global pending price
+   uint256 ticketPrice = jackpot.getDrawingState(currentDrawingId).ticketPrice;
    uint256 totalCost = ticketPrice * tickets.length;

    // ... rest of function
}
```

# [LOW-01] Delayed runJackpot Calls Enable Drawing Monopolization via Stale Timestamp Anchoring

**Summary:**
The `scaledEntropyCallback()` function in Jackpot.sol schedules the next drawing using the old drawing timestamp `(currentDrawingState.drawingTime + drawingDurationInSeconds)` instead of anchoring to the current `block timestamp`. When users delay calling `runJackpot()` beyond the scheduled drawing time (e.g., due to low participation, high gas prices, or network congestion), the entropy callback eventually executes with a stale timestamp, causing subsequent drawings to be scheduled in the past. This enables attackers to immediately lock drawings with zero tickets before legitimate users can participate, monopolize LP earnings, and create a cascading failure affecting all future drawings.

**Description:**
When `scaledEntropyCallback()` processes drawing settlement and schedules the next drawing, it uses a stale timestamp anchor based on when the drawing was supposed to start, not when `runJackpot()` was actually called:

File: `contracts/Jackpot.sol (Line 748)`
```solidity
function scaledEntropyCallback(
    uint64 sequenceNumber,
    address _providerAddress,
    bytes32 randomNumber
) external {
    // ... validation and winner selection ...
    
    // Process settlement
    JackpotLPManager(payable(lpManagerAddress)).processDrawingSettlement(
        currentDrawingId,
        numberOfWinners,
        lpEarnings,
        protocolFee
    );
    
    //  @-->>VULNERABLE: Uses old drawingTime, not when runJackpot was actually called
    _setNewDrawingState(
        newLpValue,
        currentDrawingState.drawingTime + drawingDurationInSeconds  //  Stale anchor
    );
}
```
The issue arises when there's a delay between the scheduled drawing time and when someone actually calls `runJackpot()`:

1. Drawing N scheduled for time T₀ (e.g., Monday 12:00 PM)
2. No one calls runJackpot() until T₀ + 2 hours (Monday 2:00 PM) due to:
    * Low user participation
3. Entropy callback eventually executes at T₀ + 3 hours (Monday 3:00 PM)
4. Drawing N+1 scheduled at T₀ + 1 hour = Monday 1:00 PM (2 hours in the PAST!)
   
The vulnerability exists because currentDrawingState.drawingTime represents the original scheduled time, not the actual execution time. When runJackpot() is called late, this timestamp becomes stale, but the callback still uses it as the anchor for scheduling the next drawing.

**Impact:**
A delayed runJackpot call can cause the next drawing to be scheduled in the past, enabling an attacker to immediately lock subsequent drawings with zero tickets, monopolize LP earnings, and effectively break the lottery for honest users.

**Recommended Mitigation:**
Replace implementation in `Jackpot.sol` to anchor to current time instead of scheduled time:
```diff
_setNewDrawingState(
    newLpValue,
-    currentDrawingState.drawingTime + drawingDurationInSeconds
+  Math.max(currentDrawingState.drawingTime, block.timestamp) + drawingDurationInSeconds
);
```



## QA

## [LOW-1] Integer Underflow Risk in `_applyInclusionExclusionPrinciple()`

**Module:** `TicketComboTracker.sol`
**Function:** `_applyInclusionExclusionPrinciple`

### Description

Applies the inclusion–exclusion principle to remove double-counted tickets but performs unchecked subtractions that may underflow if the tracker state becomes inconsistent.

### Vulnerability Analysis

**Root Cause:**
Unchecked arithmetic assumptions about data consistency in Solidity ≥0.8.

```solidity
function _applyInclusionExclusionPrinciple(
    Tracker storage _tracker,
    uint256[] memory _matches,
    uint256[] memory _dupMatches
) private view returns (uint256[] memory result, uint256[] memory dupResult) {
    result = new uint256[](_matches.length);
    dupResult = new uint256[](_dupMatches.length);
    
    for (uint256 k = _tracker.normalTiers; k >= 1; --k) {
        uint256 s = _matches[2*k];
        uint256 sp = _matches[2*k+1];
        uint256 sd = _dupMatches[2*k];
        uint256 sdp = _dupMatches[2*k+1];
        
        for (uint256 m = k + 1; m <= _tracker.normalTiers; ++m) {
            uint256 c = Combinations.choose(m, k);
            s -= c * result[2*m];          // ❌ unchecked subtraction
            sp -= c * result[2*m+1];       // ❌ unchecked subtraction
            sd -= c * dupResult[2*m];      // ❌ unchecked subtraction
            sdp -= c * dupResult[2*m+1];   // ❌ unchecked subtraction
        }
        
        result[2*k] = s;
        result[2*k+1] = sp;
        dupResult[2*k] = sd;
        dupResult[2*k+1] = sdp;
    }
}
```

### Exploitation Scenario

* Tracker state becomes corrupted (external bug, upgrade error)
* Higher-tier counts exceed lower-tier mathematical bounds
* Subtraction attempts to remove more than available
* Solidity 0.8 underflow protection triggers revert
* Settlement fails permanently until intervention

### Impact

| Category    | Assessment                               |
| ----------- | ---------------------------------------- |
| Severity    | **LOW**                                  |
| Likelihood  | **Very Low**                             |
| Attack Cost | Not directly exploitable                 |
| Impact      | Settlement DoS, funds temporarily locked |

### Mathematical Invariant

For all `m > k`:

```
subset_count[k] ≥ Σ(C(m,k) × unique_count[m])
```

Violation of this invariant causes underflow.

### Recommended Mitigation

Add explicit underflow checks with descriptive errors:

```solidity
for (uint256 m = k + 1; m <= _tracker.normalTiers; ++m) {
    uint256 c = Combinations.choose(m, k);

    uint256 sub = c * result[2*m];
    require(s >= sub, "Inclusion-exclusion underflow");
    s -= sub;

    sub = c * result[2*m+1];
    require(sp >= sub, "Inclusion-exclusion underflow");
    sp -= sub;

    sub = c * dupResult[2*m];
    require(sd >= sub, "Inclusion-exclusion underflow");
    sd -= sub;

    sub = c * dupResult[2*m+1];
    require(sdp >= sub, "Inclusion-exclusion underflow");
    sdp -= sub;
}
```

**Benefits**

* Clear diagnostics
* Safer failure mode
* Easier emergency recovery

---

## [LOW-2] Unbounded Gas Consumption in `toNormalsBitVector()`

**Module:** `TicketComboTracker.sol`
**Function:** `toNormalsBitVector`

### Description

Converts an array to a bit vector but lacks an upper bound on array length, allowing gas-exhaustion DoS.

### Root Cause

Missing validation on `_set.length`.

```solidity
require(_set.length != 0, "Invalid set length"); // ❌ only checks non-zero
for (uint256 i; i < _set.length; ++i) {          // ❌ unbounded loop
```

### Attack Vector

Public view function:

```solidity
function getSubsetCount(
    uint256 _drawingId,
    uint8[] memory _normals,
    uint8 _bonusball
) external view returns (TicketComboTracker.ComboCount memory)
```

### Exploitation Scenario

* Attacker submits extremely large `_normals` array
* Loop executes thousands of iterations
* Excessive gas consumption
* RPC / frontend timeouts

### Impact

| Category    | Assessment     |
| ----------- | -------------- |
| Severity    | **LOW**        |
| Likelihood  | **HIGH**       |
| Attack Cost | Minimal        |
| Impact      | View-level DoS |

### Recommended Mitigation

```solidity
require(_set.length <= 255, "Array too large");
```

---

## [LOW-3] Missing Validation in `scaledEntropyCallback`

**Module:** `Jackpot.sol`
**Function:** `scaledEntropyCallback`

### Description

Random numbers from entropy provider are downcast to `uint8` without validating ranges first.

### Risk

* Compromised entropy provider can return out-of-range values
* `toUint8()` / `toUint8Array()` reverts
* Drawing becomes permanently locked

### Impact

| Category   | Assessment                |
| ---------- | ------------------------- |
| Severity   | **LOW**                   |
| Likelihood | Low                       |
| Impact     | Drawing DoS, no fund loss |

### Recommended Mitigation

Validate entropy values **before** downcasting:

```solidity
if (_randomNumbers[0][i] < 1 || _randomNumbers[0][i] > currentDrawingState.ballMax) {
    revert JackpotErrors.InvalidRandomNumbers();
}
```

Optionally add an emergency recovery mechanism.

---

## [LOW-4] `assert()` Used Instead of `require()`

**Module:** `Combinations.sol`

### Issue

Input validation uses `assert()`:

```solidity
assert(n >= k);
assert(n <= 128);
```

### Risk

* `assert()` consumes all remaining gas
* Can be triggered by invalid admin or upstream inputs
* Causes avoidable DoS

### Recommendation

```solidity
require(n >= k, "n must be >= k");
require(n <= 128, "n too large");
```

---

## [LOW-5] Entropy Fee Refund Failure Causes DoS

**Module:** `Jackpot.sol`
**Function:** `runJackpot`

### Issue

Refunding excess ETH reverts if sender cannot receive ETH.

### Impact

Temporary DoS if malicious contracts call `runJackpot()`.

### Recommended Mitigation (Best Practice)

```solidity
if (msg.value > fee) {
    payable(msg.sender).call{value: msg.value - fee}("");
}
```

---

## [LOW-6] Missing Validation in Entropy Gas Configuration

**Module:** `Jackpot.sol`

### Issue

No validation on gas limits; arithmetic may overflow `uint32`.

```solidity
return entropyBaseGasLimit + entropyVariableGasLimit * uint32(_bonusballMax);
```

### Recommended Mitigation

* Add min/max bounds
* Use `uint256` intermediate math
* Explicit overflow check

---

## [INFO-1] `getTicketTierIds()` Returns Invalid Data for Unsettled Drawings

**Impact:** Frontend / integration confusion
**Severity:** INFORMATIONAL

### Recommendation

Return sentinel value or revert if drawing not settled.

---

## [INFO-2] `checkIfTicketsBought()` Misleading After Emergency Refunds

**Impact:** Off-chain confusion
**Severity:** INFORMATIONAL

### Recommendation

Document behavior clearly or cross-check NFT existence.

---

## [INFO-3] Missing Input Validation in `getSubsetCount()`

**Severity:** INFORMATIONAL

### Recommendation

Either:

* Add full validation
  **or**
* Explicitly document caller responsibility

---

## [INFO-4] Missing Validation in `unpackTicket()`

**Severity:** INFORMATIONAL

### Issue

Empty or invalid bit vectors cause underflow.

### Recommendation

```solidity
require(_packedTicket != 0, "Invalid packed ticket");
require(ballCount > 0, "No balls found");
```

---

## [INFO-5] Potential Out-of-Bounds Access in `_calculateBonusballOnlyMatches()`

**Severity:** INFORMATIONAL
**Risk:** Future refactors

### Recommendation

Add defensive array length checks.

---

If you want, I can also:

* Split this into **one file per finding**
* Convert to **Code4rena / Sherlock / Trail of Bits report style**
* Add **severity summary tables** or **CVSS-like scoring**

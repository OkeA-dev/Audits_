# [HIGH-01] Unauthorized Token Claim Issue in SecondSwap_StepVesting contract

**Summary:**A critical vulnerability in the `claimable()` function of the SecondSwap_StepVesting contract allows users to claim more tokens than their remaining vesting balance, potentially leading to unauthorized token extraction.

**Description:**
The `claimable()` function calculates token claims based on the original total vesting amount without properly adjusting for partial claims or token transfers. This creates a discrepancy between the calculated claimable amount and the actual remaining vesting balance.

**Impact:**Loss of users' fund

**Recommended Mitigation:**
To mitigate this issue, consider modifying the `claimable()`:

```diff
 function claimable(address _beneficiary) public view returns (uint256, uint256) { // to check if when the token is claimable.
        Vesting memory vesting = _vestings[_beneficiary];
        if (vesting.totalAmount == 0) {
            return (0, 0);
        }

        uint256 currentTime = Math.min(block.timestamp, endTime);
        if (currentTime < startTime) { 
            return (0, 0);
        }

        uint256 elapsedTime = currentTime - startTime;
        uint256 currentStep = elapsedTime / stepDuration; 
        uint256 claimableSteps = currentStep - vesting.stepsClaimed;

        uint256 claimableAmount;

        if (vesting.stepsClaimed + claimableSteps >= numOfSteps) {
            //[BUG FIX] user can buy more than they are allocated
            claimableAmount = vesting.totalAmount - vesting.amountClaimed;
            return (claimableAmount, claimableSteps);
        }
-        claimableAmount = vesting.releaseRate * claimableSteps;
+        claimableAmount = (vesting.totalAmount - vesting.amountClaimed)/ (numOfSteps - vesting.stepsClaimed); // (60,000 - 50,000) / (4 - 2) = 10,000/2 = 5,000.
        return (claimableAmount, claimableSteps);
    }
```

# [MEDUIM-01] Lack of Discount Percentage Validation
**Description:**
The current implementation of the _getDiscountedPrice() function lacks explicit validation for the discount percentage, allowing potentially invalid or excessive discount percentages to be processed without proper checks. This vulnerability enables the input of discount percentages that could lead to unexpected or mathematically incorrect pricing calculations.
```solidity
   function _getDiscountedPrice(Listing storage listing, uint256 _amount) private view returns (uint256) {
        uint256 discountedPrice = listing.pricePerUnit;

        if (listing.discountType == DiscountType.LINEAR) {
            discountedPrice = (discountedPrice * (BASE - ((_amount * listing.discountPct) / listing.total))) / BASE;
        } else if (listing.discountType == DiscountType.FIX) {
            discountedPrice = (discountedPrice * (BASE - listing.discountPct)) / BASE;
        }
        return discountedPrice;
    }
```
**Impact:**
Unchecked discount percentages could result in arbitrary price manipulations, potentially causing financial inconsistencies or unintended pricing behaviors.
Proof of Concept

Demonstrate ability to set discount percentages beyond 100% of the base value
Show scenarios where discount calculations produce negative or zero prices
Highlight potential exploitation of unbounded discount percentage inputs
Illustrate how malicious actors could manipulate pricing through extreme discount values

**Recommended Mitigation:**
Protocol Specific
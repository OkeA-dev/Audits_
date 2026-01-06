# [HIGH-01] Buyer fee-rate used for sell trades, causing sellers under/over‑paid and treasury misaccounted

**Summary:**
The contract calculates trade fees using the buyer’s effective trade-fee rate but subtracts that fee from the seller’s proceeds, causing sellers to be charged according to the buyer’s tier.
**Description:**
* Where it happens: `_executeTokenSwap` and the buy-branches of `_executeAgainstMatcher` compute

    * `paymentAmount = (fillAmount * sellPrice) / 10000`
    * `tradeFee = paymentAmount * _getEffectiveTradeFeeRate(buyer) / 10000`
    * `netPayment = paymentAmount - tradeFee`
    * then transfer tradeFee → treasury and netPayment → seller.

* Why this is wrong: For a seller-exit the seller should bear the trade fee (or the protocol must explicitly charge the buyer on-top ). Using the buyer’s fee-rate misattributes the fee and ties seller receipts to buyer-tier settings.

**Scenerio**
Trade: paymentAmount = $650

* Case A (buyer lower than seller):

    * buyerRate = 0.25% → fee = 1.625;

    * sellerRate = 1.00 1.625; sellerRate = 1.00 6.50

    * Current code: seller receives 648.375

        * (undercharged by 648.375 (undercharged by 4.125; treasury short by 4.875 vs correct 4.875 vs correct 6.50))

* Case B (buyer higher than seller):

    * buyerRate = 2.00% → fee = 13.00;

    * sellerRate = 0.50 13.00; sellerRate = 0.50 3.25

    * Current code: seller receives 637.00

        * (seller loses 637.00 (seller loses 9.75 vs correct $646.75))

* Scale:

    * Mismatch scales linearly; at 1,000,000 payment

    * a 150 bpm mismatch => 1,000,000 payment a 150 bpm mismatch => 15,000 incorrect movement

**Impact:**
Sellers can receive materially incorrect proceeds and the treasury’s fee accounting is unreliable

**Recommended Mitigation:**
Fix code to charge the seller’s rate for sell executions :
```diff
        // Calculate trade fee from buyer
  
        // Calculate trade fee from seller  
+        uint256 sellerFeeRate = _getEffectiveTradeFeeRate(sellOrder.user);  
+        uint256 tradeFee = (paymentAmount * sellerFeeRate) / 10000;  
+        uint256 netPayment = paymentAmount - tradeFee;  
```
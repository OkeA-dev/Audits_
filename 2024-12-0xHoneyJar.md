# [LOW-01] User Can Deny Referral Their Fee Share Due to Rounding Issue
**Summary**
A rounding issue allows users to withhold the referral's share of protocol fees.

**Description:**
When the owner of a HoneyLocker withdraws funds using the `withdrawERC20` or `withdrawBERA` methods, a `2%` fee is charged, which is split between the referral (`30%` of the fee) and the protocol treasury (`70%` of the fee).

However, due to the EVM rounding down calculations, the owner can exploit this to prevent the referral from receiving their share of the fee. The issue stems from the implementation of Beekeeper::distributeFee.
```solidity
uint256 public standardReferrerFeeShare = 3000; // in bps 30%

         function referrerFeeShare(address _referrer) public view returns (uint256) {
        return _referrerFeeShare[_referrer] != 0 ? _referrerFeeShare[_referrer] : standardReferrerFeeShare;
         }

        function distributeFees(address _referrer, address _token, uint256 _amount) external payable {
        bool isBera = _token == address(0);
        if (!isBera && _token.code.length == 0) revert NoCodeForToken();
        // if not an authorized referrer, send everything to treasury
        if (!isReferrer[_referrer]) {
            isBera ? STL.safeTransferETH(treasury, _amount) : STL.safeTransfer(_token, treasury, _amount);
            emit FeesDistributed(treasury, _token, _amount);
            return;
        }
        // use the referrer fee override if it exists, otherwise use the original referrer
        address referrer = referrerOverrides[_referrer] != address(0) ? referrerOverrides[_referrer] : _referrer;
        uint256 referrerFeeShareInBps = referrerFeeShare(referrer);
@>        uint256 referrerFee = (_amount * referrerFeeShareInBps) / 10000; 

        if (isBera) {
            STL.safeTransferETH(referrer, referrerFee);
            STL.safeTransferETH(treasury, _amount - referrerFee);
        } else {
            STL.safeTransfer(_token, referrer, referrerFee);
            STL.safeTransfer(_token, treasury, _amount - referrerFee);
        }

        emit FeesDistributed(referrer, _token, referrerFee);
        emit FeesDistributed(treasury, _token, _amount - referrerFee);
    }
```
To exploit this, the owner can withdraw funds in very small batches. For example, if an owner wants to withdraw 2,500,000 tokens, they can repeatedly request withdrawals of just 150 tokens per transaction. This specific amount is chosen because, at a nominal fee rate of 30% from the 2% fee, the calculation rounds down to zero for the referral's share, effectively depriving them of their portion.

**Impact:**
Referral won't get paid.

**Recommended Mitigation:**
To mitigate the issue modify the fee calculation in the `distributeFee` to alway round up, ensuring that no withdrawal is completed without proper fee distribution.
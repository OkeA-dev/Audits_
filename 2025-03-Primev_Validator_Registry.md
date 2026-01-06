# [HIGH-01] Unauthorized Override Assignment Enables Arbitrary Fund Theft from Unclaimed Rewards

**Summary:**The `overrideReceiver()` function lacks access control validation, allowing any address to designate another address as their override and subsequently drain that address's unclaimed rewards through the migration mechanism in `removeOverrideAddress()`.

**Description:** 
The overrideReceiver() function in RewardManager.sol fails to validate that:

* The caller (msg.sender) has legitimate authority to set an override
* The target override address (overrideAddress) has consented to the relationship

```solidity
function overrideReceiver(address overrideAddress, bool migrateExistingRewards) external whenNotPaused nonReentrant {
    if (migrateExistingRewards) { _migrateRewards(msg.sender, overrideAddress); }
    require(overrideAddress != address(0) && overrideAddress != msg.sender, InvalidAddress());
    overrideAddresses[msg.sender] = overrideAddress; // ❌ No authorization check
    emit OverrideAddressSet(msg.sender, overrideAddress);
}
```

**Impact:**Any attacker can steal the complete unclaimed reward balance from any address in the system without requiring any legitimate relationship or authorization from the victim.

**Recommended Mitigation:**
Primary Fix: Access Control Validation
```solidity
function overrideReceiver(address overrideAddress, bool migrateExistingRewards) external whenNotPaused nonReentrant {
    // Validate caller is legitimate receiver for at least one pubkey
    require(_isLegitimateReceiver(msg.sender), "Caller not authorized to set overrides");
    
    if (migrateExistingRewards) { _migrateRewards(msg.sender, overrideAddress); }
    require(overrideAddress != address(0) && overrideAddress != msg.sender, InvalidAddress());
    overrideAddresses[msg.sender] = overrideAddress;
    emit OverrideAddressSet(msg.sender, overrideAddress);
}

// Helper function to validate legitimate receivers
function _isLegitimateReceiver(address addr) internal view returns (bool) {
    // Check if address receives rewards for any registered pubkey across all registries
    // Implementation depends on registry interfaces
}
```
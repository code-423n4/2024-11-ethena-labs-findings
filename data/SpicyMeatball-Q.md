### L-01 `ustb` address is not set upon `UStbMinting.sol` creation

When the `UStbMinting.sol` contract is created, certain validation checks are performed to prevent invalid configurations, as shown here:

https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L187

```solidity
        for (uint128 k = 0; k < _tokenConfig.length; ) {
>>          if (tokenConfig[_assets[k]].isActive || _assets[k] == address(0) || _assets[k] == address(ustb)) {
                revert InvalidAssetAddress();
```

https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L176

```solidity
    function addCustodianAddress(address custodian) public onlyRole(DEFAULT_ADMIN_ROLE) {
>>      if (custodian == address(0) || custodian == address(ustb) || !_custodianAddresses.add(custodian)) {
            revert InvalidCustodianAddress();
        }
```

In both checks, `address(ustb)` is referenced, yet during the contract’s creation, `ustb` is not initialized and defaults to `address(0)`. This results in incomplete checks, as `ustb` does not yet hold the intended address value.

Consider setting `ustb` address in the constructor.
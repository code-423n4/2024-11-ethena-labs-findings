| Issue ID | Issue Name                                                                 |
|----------|----------------------------------------------------------------------------|
| [L-01](#l-01-the-addblacklistaddress-and-addwhitelistaddress-functions-do-not-check-whether-the-user-has-opposite-role) | The `addBlacklistAddress` and `addWhitelistAddress` functions do not check whether the user has opposite role |
| [L-02](#l-02-in-the-constructor-of-ustbminting-contract-it-does-not-set-ustb) | In the constructor of `UStbMinting` contract, it does not set `ustb` |
| [L-03](#l-03-the-_computedomainseparator-function-incorrectly-encodes-bytes32-variable-as-string-type) | The `_computeDomainSeparator` function incorrectly encodes `bytes32` variable as `string` type |
| [L-04](#l-04-differenceinbps-is-calculated-with-a-precision-of-104) | `differenceInBps` is calculated with a precision of 10^4 |
| [L-05](#l-05-if-the-collateral_asset-is-native-token-minting-is-unavailable-even-though-redeeming-is-available) | If the `collateral_asset` is native token, minting is unavailable even though redeeming is available. |
| [L-06](#l-06-most-of-the-event-parameters-are-of-the-uint256-data-type-but-uint128-variables-are-used-when-emitting-them) | Most of the event parameters are of the `uint256` data type, but `uint128` variables are used when emitting them |
| [L-07](#l-07-thegatekeeper_role-shouldnt-be-allowed-to-remove-the-collateral_manager_role) | The`GATEKEEPER_ROLE` shouldn't be allowed to remove the `COLLATERAL_MANAGER_ROLE` |
| [L-08](#l-08-unnecessary-check-of-tokenconfigassetisactive-in-the-_transfertobeneficiary-function) | Unnecessary check of `tokenConfig[asset].isActive` in the `_transferToBeneficiary` function. |
| [L-09](#l-09-non-blacklisted-addresses-cant-burn-ustb-tokens-in-a-fully_enabled-transfer-state-if-address0-is-blaklisted) | Non-blacklisted addresses can't burn `UStb` tokens in a `FULLY_ENABLED` transfer state if `address(0)` is blaklisted |
| [L-10](#l-10-the-redistributelockedamount-function-does-not-verify-if-the-address-to-possesses-the-whitelisted_role-when-whitelist_enabled) | The `redistributeLockedAmount()` function does not verify if the address `to` possesses the `WHITELISTED_ROLE` when `WHITELIST_ENABLED`. |
| [L-11](#l-11-there-is-no-function-available-to-transfer-ustb-from-unwhitelisted-users-to-whitelisted-users) | There is no function available to transfer UStb from unwhitelisted users to whitelisted users. |
| [L-12](#l-12-the-_beforetokentransfer-function-does-not-verify-whether-addresses-are-whitelisted-when-whitelist_enabled-is-set) | The `_beforeTokenTransfer()` function does not verify whether addresses are whitelisted when `WHITELIST_ENABLED` is set. |

## [L-01] The `addBlacklistAddress` and `addWhitelistAddress` functions do not check whether the user has opposite role

The [`addBlacklistAddress`](https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStb.sol#L73) function does not check whether the user has `WHITELISTED_ROLE`

```solidity
    function addBlacklistAddress(address[] calldata users) external onlyRole(BLACKLIST_MANAGER_ROLE) {
        for (uint8 i = 0; i < users.length; i++) {
            _grantRole(BLACKLISTED_ROLE, users[i]);
        }
    }
```

The [`addWhitelistAddress`](https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStb.sol#L91) function does not check whether the user has `BLACKLISTED_ROLE`

```solidity
    function addWhitelistAddress(address[] calldata users) external onlyRole(WHITELIST_MANAGER_ROLE) {
        for (uint8 i = 0; i < users.length; i++) {
            _grantRole(WHITELISTED_ROLE, users[i]);
        }
    }
```

If `WHITELIST_MANAGER_ROLE` grants `WHITELISTED_ROLE` to blacklisted user and does not remove BLACKLISTED_ROLE, the user cannot transfer `UStb` token in `WHITELIST_ENABLED` state from [L205](https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStb.sol#L206).

```solidity
        } else if (
            hasRole(WHITELISTED_ROLE, msg.sender) &&
            hasRole(WHITELISTED_ROLE, from) &&
            hasRole(WHITELISTED_ROLE, to) &&
            !hasRole(BLACKLISTED_ROLE, msg.sender) &&
L206:       !hasRole(BLACKLISTED_ROLE, from) &&
            !hasRole(BLACKLISTED_ROLE, to)
        ) {
            // n.b. an address can be whitelisted and blacklisted at the same time
            // normal case
        } else {
            revert OperationNotAllowed();
        }
```

It is recommended to changed the code as following.

```diff
    function addBlacklistAddress(address[] calldata users) external onlyRole(BLACKLIST_MANAGER_ROLE) {
        for (uint8 i = 0; i < users.length; i++) {
+           if (hasRole(WHITELISTED_ROLE, users[i]))
+               _revokeRole(WHITELISTED_ROLE, users[i]);
            _grantRole(BLACKLISTED_ROLE, users[i]);
        }
    }
    
    function addWhitelistAddress(address[] calldata users) external onlyRole(WHITELIST_MANAGER_ROLE) {
        for (uint8 i = 0; i < users.length; i++) {
+           if (hasRole(BLACKLISTED_ROLE, users[i]))
+               _revokeRole(BLACKLISTED_ROLE, users[i]);
            _grantRole(WHITELISTED_ROLE, users[i]);
        }
    }
```

## [L-02] In the constructor of `UStbMinting` contract, it does not set `ustb`

The constructor of `UStbMinting` contract does not set `ustb`, it checks whether `_assets[k]` is `address(ustb)` from L187

https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L187

```solidity
L187:   if (tokenConfig[_assets[k]].isActive || _assets[k] == address(0) || _assets[k] == address(ustb)) {
            revert InvalidAssetAddress();
        }
```

Because initial value `ustb` is 0, `_assets[k] == address(ustb)` is same as `_assets[k] == address(0)`.
This means `_assets[k] == address(ustb)` is unnecessary check.

Sets the initial `ustb` in the constructor of the `UStbMinting` contract.


## [L-03] The `_computeDomainSeparator` function incorrectly encodes `bytes32` variable as `string` type

The `EIP712_DOMAIN`'s `name` and `version` parameters are `string` type, and `EIP_712_NAME` and `EIP712_REVISION` are `bytes32` type.

```solidity
L28:    bytes32 private constant EIP712_DOMAIN =
            keccak256("EIP712Domain(string name,string version,uint256 chainId,address verifyingContract)");
L56:    bytes32 private constant EIP_712_NAME = keccak256("EthenaUStbMinting");
L59:    bytes32 private constant EIP712_REVISION = keccak256("1");
```

The [_computeDomainSeparator](https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L665) function encodes with inconsistent data type.
```solidity
        function _computeDomainSeparator() internal view returns (bytes32) {
L665:       return keccak256(abi.encode(EIP712_DOMAIN, EIP_712_NAME, EIP712_REVISION, block.chainid, address(this)));
        }
```
Ensure that the data types of the parameters `EIP712_DOMAIN`, `EIP_712_NAME`, and `EIP712_REVISION` match.

## [L-04] `differenceInBps` is calculated with a precision of 10^4

Since `STABLES_RATIO_MULTIPLIER` is 10,000, `differenceInBps` is calculated with a precision of 10^4.

https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L568

```solidity
L568:     uint128 differenceInBps = (difference * STABLES_RATIO_MULTIPLIER) / ustbAmount;
```

It needs to calculate it with higher precision.

Sets the `STABLES_RATIO_MULTIPLIER` as higher value like `1e18`.

## [L-05] If the `collateral_asset` is native token, minting is unavailable even though redeeming is available.

In the `_transferCollateral` function, if the `collateral_asset` is native token, it reverts. This causes minting reverts.

```solidity
File: contracts\ustb\UStbMinting.sol
608:         if (!tokenConfig[asset].isActive || asset == NATIVE_TOKEN) revert UnsupportedAsset();
```

However, redeeming is available even though the `collateral_asset` is native token.

```solidity
    function _transferToBeneficiary(address beneficiary, address asset, uint128 amount) internal {
L:588:  if (asset == NATIVE_TOKEN) {
            if (address(this).balance < amount) revert InvalidAmount();
            (bool success, ) = (beneficiary).call{value: amount}("");
            if (!success) revert TransferFailed();
        } else {
            if (!tokenConfig[asset].isActive) revert UnsupportedAsset();
            IERC20(asset).safeTransfer(beneficiary, amount);
        }
    }
```

Improve the minting mechanism to allow native token.

## [L-06] Most of the event parameters are of the `uint256` data type, but `uint128` variables are used when emitting them

The `collateral_amount` of the `Mint` event are uint256 data type.

https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/IUStbMintingEvents.sol#L17

```solidity
    event Mint(
        string indexed order_id,
        address indexed benefactor,
        address indexed beneficiary,
        address minter,
        address collateral_asset,
L17:    uint256 collateral_amount,
        uint256 ustb_amount
    );
```

However `order.collateral_amount` is `uint128` data type in the `mint` function.

```solidity
        emit Mint(
            order.order_id,
            order.benefactor,
            order.beneficiary,
            msg.sender,
            order.collateral_asset,
L251:       order.collateral_amount,
            order.ustb_amount
        );
```

Ensure that the data types of the events and variables match.

## [L-07] The`GATEKEEPER_ROLE` shouldn't be allowed to remove the `COLLATERAL_MANAGER_ROLE`

From the [readme](https://github.com/code-423n4/2024-11-ethena-labs/blob/main/README.md), the `GATEKEEPER` has the ability to disable minting and redeeming.
| Role              | Description                                                                        |
|-------------------|------------------------------------------------------------------------------------|
| GATEKEEPER        | has the ability to disable minting and redeeming                                   |

However, in the codebase, the `GATEKEEPER_ROLE` can remove the `COLLATERAL_MANAGER_ROLE` from an account.
```solidity
File: contracts\ustb\UStbMinting.sol
379:     function removeCollateralManagerRole(address collateralManager) external onlyRole(GATEKEEPER_ROLE) {
380:         _revokeRole(`COLLATERAL_MANAGER_ROLE`, collateralManager);
381:     }
```
The `COLLATERAL_MANAGER_ROLE` can transfer an asset to a custody wallet using the `transferToCustody()` function.

It means that the `GATEKEEPER_ROLE` can remove the functionality to transfer assets to the custody wallet.

It is recommended to modify the code as follows:
```diff
File: contracts\ustb\UStbMinting.sol
-        function removeCollateralManagerRole(address collateralManager) external onlyRole(DEFAULT_ADMIN_ROLE) {
+        function removeCollateralManagerRole(address collateralManager) external onlyRole(GATEKEEPER_ROLE) {
380:         _revokeRole(`COLLATERAL_MANAGER_ROLE`, collateralManager);
381:     }
```

## [L-08] Unnecessary check of `tokenConfig[asset].isActive` in the `_transferToBeneficiary` function.

The `redeem` function has `belowMaxRedeemPerBlock` modifier and it checks `_config.isActive` is true from [L127](https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L127).

```solidity
L127:        if (!_config.isActive) revert UnsupportedAsset();
```

The `redeem` function calls `_transferToBeneficiary` and it also checks `tokenConfig[asset].isActive` is true from L594.

```solidity
L594:            if (!tokenConfig[asset].isActive) revert UnsupportedAsset();
```

This is unnecessary double check. It is recommended to remove the check.

## [L-09] Non-blacklisted addresses can't burn `UStb` tokens in a `FULLY_ENABLED` transfer state if `address(0)` is blaklisted

From the [Main invariants](https://github.com/code-423n4/2024-11-ethena-labs/blob/main/README.md):
> ### Main invariants
> - Only non-blacklisted addresses can send/receive/burn UStb tokens in a `FULLY_ENABLED` transfer state.

However, if `address(0)` is blacklisted, non-blacklisted addresses can't burn their tokens, which violates the main invariants.

It is recommended to modify the code as follows:
```diff
File: contracts\ustb\UStb.sol
176:             ) {
177:                 // redistributing - mint
+                } else if (
+                    !hasRole(BLACKLISTED_ROLE, msg.sender) && !hasRole(BLACKLISTED_ROLE, from) && to == address(0)
+                ) {
+                    // burn
178:             } else if (
179:                 !hasRole(BLACKLISTED_ROLE, msg.sender) &&
```

## [L-10] The `redistributeLockedAmount()` function does not verify if the address `to` possesses the `WHITELISTED_ROLE` when `WHITELIST_ENABLED`.

When the state is `WHITELIST_ENABLED`, only addresses with the `WHITELISTED_ROLE` are permitted to receive UStb. However, in the `redistributeLockedAmount()` function, the only check performed is to ensure that the address to does not have the `BLACKLISTED_ROLE`. Consequently, redistribution to an unwhitelisted address is allowed.

https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L111-L120

```solidity
    function redistributeLockedAmount(address from, address to) external nonReentrant onlyRole(DEFAULT_ADMIN_ROLE) {
112     if (hasRole(BLACKLISTED_ROLE, from) && !hasRole(BLACKLISTED_ROLE, to)) {
            uint256 amountToDistribute = balanceOf(from);
            _burn(from, amountToDistribute);
            _mint(to, amountToDistribute);
            emit LockedAmountRedistributed(from, to, amountToDistribute);
        } else {
            revert OperationNotAllowed();
        }
    }
```

Verify whether the address to has the `WHITELISTED_ROLE` when the state is `WHITELIST_ENABLED`.

## [L-11] There is no function available to transfer UStb from unwhitelisted users to whitelisted users.

The `redistributeLockedAmount()` function facilitates the transfer of UStb from a blacklisted user to an unblacklisted user. However, when the state is `WHITELIST_ENABLED`, there is no mechanism in place to transfer UStb from an unwhitelisted user to a whitelisted user.

https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L111-L120

```solidity
    function redistributeLockedAmount(address from, address to) external nonReentrant onlyRole(DEFAULT_ADMIN_ROLE) {
112     if (hasRole(BLACKLISTED_ROLE, from) && !hasRole(BLACKLISTED_ROLE, to)) {
            uint256 amountToDistribute = balanceOf(from);
            _burn(from, amountToDistribute);
            _mint(to, amountToDistribute);
            emit LockedAmountRedistributed(from, to, amountToDistribute);
        } else {
            revert OperationNotAllowed();
        }
    }
```

Include a function to transfer UStb from an unwhitelisted user to a whitelisted user when the state is `WHITELIST_ENABLED`.

## [L-12] The `_beforeTokenTransfer()` function does not verify whether addresses are whitelisted when `WHITELIST_ENABLED` is set.

As noted in line 191, the function only checks if the address `to` is not blacklisted. Consequently, unwhitelisted users can still receive UStb when `WHITELIST_ENABLED` is active. This issue appears in several locations.

https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStb.sol#L165-L218

```solidity
    function _beforeTokenTransfer(address from, address to, uint256) internal virtual override {
        // State 2 - Transfers fully enabled except for blacklisted addresses
        if (transferState == TransferState.FULLY_ENABLED) {

            ...

        } else if (transferState == TransferState.WHITELIST_ENABLED) {
            if (hasRole(MINTER_CONTRACT, msg.sender) && !hasRole(BLACKLISTED_ROLE, from) && to == address(0)) {
                // redeeming
191         } else if (hasRole(MINTER_CONTRACT, msg.sender) && from == address(0) && !hasRole(BLACKLISTED_ROLE, to)) {
                // minting
            } else if (hasRole(DEFAULT_ADMIN_ROLE, msg.sender) && hasRole(BLACKLISTED_ROLE, from) && to == address(0)) {
                // redistributing - burn
            } else if (
                hasRole(DEFAULT_ADMIN_ROLE, msg.sender) && from == address(0) && !hasRole(BLACKLISTED_ROLE, to)
            ) {
                // redistributing - mint
            } else if (hasRole(WHITELISTED_ROLE, msg.sender) && hasRole(WHITELISTED_ROLE, from) && to == address(0)) {
                // whitelisted user can burn
            } else if (
                hasRole(WHITELISTED_ROLE, msg.sender) &&
                hasRole(WHITELISTED_ROLE, from) &&
                hasRole(WHITELISTED_ROLE, to) &&
                !hasRole(BLACKLISTED_ROLE, msg.sender) &&
                !hasRole(BLACKLISTED_ROLE, from) &&
                !hasRole(BLACKLISTED_ROLE, to)
            ) {
                // n.b. an address can be whitelisted and blacklisted at the same time
                // normal case
            } else {
                revert OperationNotAllowed();
            }
            // State 0 - Fully disabled transfers
        } else if (transferState == TransferState.FULLY_DISABLED) {
            revert OperationNotAllowed();
        }
    }
```

Include checks to ensure that the addresses are whitelisted.

| Count | Title |
| --- | --- |
| [QA-01](#qa-01-error-in-stable-limit-verification) | Error in Stable Limit Verification |
| [QA-02](#qa-02-...) | Total Mint and Redeem is not Reset which will Allow Griefing Attack |
| [QA-03](#qa-03-...) | Incomplete Order Verification |
| [QA-04](#qa-04-...) | Token Activeness Validation Absence |
| [QA-05](#qa-05-...) | Whitelisted Address DOS Before Token Transfer |

| Total Issues | 5 |
| --- | --- |

## [QA-01] Error in Stable Limit Verification

##### **Issue Description:**
Unstable collateralAmount to ustbAmount difference would go through during redeem and minting due to oversight in implementation
https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L571
https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L573
```solidity
function verifyStablesLimit(
        uint128 collateralAmount,
        uint128 ustbAmount,
        address collateralAsset,
        OrderType orderType
    ) public view returns (bool) {
        uint128 ustbDecimals = _getDecimals(address(ustb));
        uint128 collateralDecimals = _getDecimals(collateralAsset);

        uint128 normalizedCollateralAmount;
        uint128 scale = uint128(
            ustbDecimals > collateralDecimals
                ? 10 ** (ustbDecimals - collateralDecimals)
                : 10 ** (collateralDecimals - ustbDecimals)
        );

        normalizedCollateralAmount = ustbDecimals > collateralDecimals
            ? collateralAmount * scale
            : collateralAmount / scale;

        uint128 difference = normalizedCollateralAmount > ustbAmount
            ? normalizedCollateralAmount - ustbAmount
            : ustbAmount - normalizedCollateralAmount;

        uint128 differenceInBps = (difference * STABLES_RATIO_MULTIPLIER) / ustbAmount;

        if (orderType == OrderType.MINT) {
>>>            return ustbAmount > normalizedCollateralAmount ? differenceInBps <= stablesDeltaLimit : true;
        } else {
>>>            return normalizedCollateralAmount > ustbAmount ? differenceInBps <= stablesDeltaLimit : true;
        }
    }
```
The code above shows how stable limit is verified in the relationship between collateral and ustbamount during minting and redeem, the verification is to ensure the interaction input values are not far apart, however the protocol made an exception as noted in the two pointers in code above and simply return true regardless of difference value, this is because amount lost in this case would be at the expense of the caller and not the protocol but this should still be corrected the caller is a part of the ethena community and stable limit should be verified to prevent fund loss for users
##### **Recommendation:**
As adjusted below regardless of if normalizedCollateralAmount is less than or greater than ustbAmount differenceInBps should be validated to ensure it is not above stable delta limit
```solidity
function verifyStablesLimit(
        uint128 collateralAmount,
        uint128 ustbAmount,
        address collateralAsset,
        OrderType orderType
    ) public view returns (bool) {
        uint128 ustbDecimals = _getDecimals(address(ustb));
        uint128 collateralDecimals = _getDecimals(collateralAsset);

        uint128 normalizedCollateralAmount;
        uint128 scale = uint128(
            ustbDecimals > collateralDecimals
                ? 10 ** (ustbDecimals - collateralDecimals)
                : 10 ** (collateralDecimals - ustbDecimals)
        );

        normalizedCollateralAmount = ustbDecimals > collateralDecimals
            ? collateralAmount * scale
            : collateralAmount / scale;

        uint128 difference = normalizedCollateralAmount > ustbAmount
            ? normalizedCollateralAmount - ustbAmount
            : ustbAmount - normalizedCollateralAmount;

        uint128 differenceInBps = (difference * STABLES_RATIO_MULTIPLIER) / ustbAmount;

        if (orderType == OrderType.MINT) {
---            return ustbAmount > normalizedCollateralAmount ? differenceInBps <= stablesDeltaLimit : true;
+++            return  differenceInBps <= stablesDeltaLimit ;
        } else {
---            return normalizedCollateralAmount > ustbAmount ? differenceInBps <= stablesDeltaLimit : true;
+++            return  differenceInBps <= stablesDeltaLimit ;
        }
    }
```

## [QA-02] Total Mint and Redeem is not Reset which will Allow Griefing Attack
##### **Issue Description:**
https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L227-L277
https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L116
https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L128
```solidity
   function mint(
        Order calldata order,
        Route calldata route,
        Signature calldata signature
    )
        external
        override
        nonReentrant
        onlyRole(MINTER_ROLE)
>>>        belowMaxMintPerBlock(order.ustb_amount, order.collateral_asset)
        belowGlobalMaxMintPerBlock(order.ustb_amount)
    {
        if (order.order_type != OrderType.MINT) revert InvalidOrder();
        verifyOrder(order, signature);
        if (!verifyRoute(route)) revert InvalidRoute();
        _deduplicateOrder(order.benefactor, order.nonce);
        // Add to the minted amount in this block
>>>        totalPerBlockPerAsset[block.number][order.collateral_asset].mintedPerBlock += order.ustb_amount;
        totalPerBlock[block.number].mintedPerBlock += order.ustb_amount;
...
    }
...
    function redeem(
        Order calldata order,
        Signature calldata signature
    )
        external
        override
        nonReentrant
        onlyRole(REDEEMER_ROLE)
>>>        belowMaxRedeemPerBlock(order.ustb_amount, order.collateral_asset)
        belowGlobalMaxRedeemPerBlock(order.ustb_amount)
    {
        if (order.order_type != OrderType.REDEEM) revert InvalidOrder();
        verifyOrder(order, signature);
        _deduplicateOrder(order.benefactor, order.nonce);
        // Add to the redeemed amount in this block
>>>        totalPerBlockPerAsset[block.number][order.collateral_asset].redeemedPerBlock += order.ustb_amount;
        totalPerBlock[block.number].redeemedPerBlock += order.ustb_amount;
  ...
    }
```
From the mint and redeem code above it can be noted that input amount is accumulated regardless of minting or redeem, it can also be noted that both functions have maximum limitations to ensure caller does not over mint or over redeem. Problem is that a bad actor/Minter can continuously mint and redeem at intervals until the max limit is reach which will disenfranchise other users who want to mint or redeem
```solidity
    modifier belowMaxMintPerBlock(uint128 mintAmount, address asset) {
        TokenConfig memory _config = tokenConfig[asset];
        if (!_config.isActive) revert UnsupportedAsset();
>>>        if (totalPerBlockPerAsset[block.number][asset].mintedPerBlock + mintAmount > _config.maxMintPerBlock) {
            revert MaxMintPerBlockExceeded();
        }
        _;
    }
...
    modifier belowMaxRedeemPerBlock(uint128 redeemAmount, address asset) {
        TokenConfig memory _config = tokenConfig[asset];
        if (!_config.isActive) revert UnsupportedAsset();
        if (totalPerBlockPerAsset[block.number][asset].redeemedPerBlock + redeemAmount > _config.maxRedeemPerBlock) {
            revert MaxRedeemPerBlockExceeded();
        }
        _;
    }
```
lets assume max mint and redeem per block is set to 1000
minter mint 100 token at first
then redeems it, then mint what was redeemed again
and continuously repeat this process until the maximum limit is reached, other users would not be able to mint or redeem again as the maximum limit would have been reached. 
Note: The bad actor has no benefit from this call but rather just to break protocol proper functionality which should be prevented
##### **Recommendation:**
As adjusted below protocol should keep track of mint and redeem by reducing mint value when user calls redeem instead of handling them independently.
```solidity
    function redeem(
        Order calldata order,
        Signature calldata signature
    )
        external
        override
        nonReentrant
        onlyRole(REDEEMER_ROLE)
        belowMaxRedeemPerBlock(order.ustb_amount, order.collateral_asset)
        belowGlobalMaxRedeemPerBlock(order.ustb_amount)
    {
        if (order.order_type != OrderType.REDEEM) revert InvalidOrder();
        verifyOrder(order, signature);
        _deduplicateOrder(order.benefactor, order.nonce);
        // Add to the redeemed amount in this block
---        totalPerBlockPerAsset[block.number][order.collateral_asset].redeemedPerBlock += order.ustb_amount;
+++        totalPerBlockPerAsset[block.number][order.collateral_asset].mintedPerBlock -= order.ustb_amount;
        totalPerBlock[block.number].redeemedPerBlock += order.ustb_amount;
  ...
    }
```

## [QA-03] Incomplete Order Verification
##### **Issue Description:**
https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L490
```solidity
 function verifyOrder(
        Order calldata order,
        Signature calldata signature
    ) public view override returns (bytes32 taker_order_hash) {
        ...
        TokenType typeOfToken = tokenConfig[order.collateral_asset].tokenType;
        if (typeOfToken == TokenType.STABLE) {
            if (
                !verifyStablesLimit(
                    order.collateral_amount,
                    order.ustb_amount,
                    order.collateral_asset,
                    order.order_type
                )
            ) {
                revert InvalidStablePrice();
            }
        }
        if (order.beneficiary == address(0)) revert InvalidAddress();
        if (order.collateral_amount == 0 || order.ustb_amount == 0) revert InvalidAmount();
        if (block.timestamp > order.expiry) revert SignatureExpired();
    }
```
The code above shows how verification is done in the UStbMinting contract, even though there is validation to ensure order.collateral_asset set by admin cant be address zero, there is an oversight in the verifyOrder function to ensure order.collateral_asset used in verification is not zero address which makes the verification incomplete
##### **Recommendation:**
As adjusted below collateral asset should be verified before usage
```solidity
 function verifyOrder(
        Order calldata order,
        Signature calldata signature
    ) public view override returns (bytes32 taker_order_hash) {
        ...
+++   require( order.collateral_asset != address(0) , "Asset Error" )
        TokenType typeOfToken = tokenConfig[order.collateral_asset].tokenType;
        if (typeOfToken == TokenType.STABLE) {
            if (
                !verifyStablesLimit(
                   ...
                )
            ) {
                revert InvalidStablePrice();
            }
        }
        if (order.beneficiary == address(0)) revert InvalidAddress();
        if (order.collateral_amount == 0 || order.ustb_amount == 0) revert InvalidAmount();
        if (block.timestamp > order.expiry) revert SignatureExpired();
    }
```
## [QA-04] Token Activeness Validation Absence
##### **Issue Description:**
https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L647
https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStbMinting.sol#L658
```solidity
    function _setMaxMintPerBlock(uint128 _maxMintPerBlock, address asset) internal {
        uint128 oldMaxMintPerBlock = tokenConfig[asset].maxMintPerBlock;
        tokenConfig[asset].maxMintPerBlock = _maxMintPerBlock;
        emit MaxMintPerBlockChanged(oldMaxMintPerBlock, _maxMintPerBlock, asset);
    }
...
    function _setMaxRedeemPerBlock(uint128 _maxRedeemPerBlock, address asset) internal {
        uint128 oldMaxRedeemPerBlock = tokenConfig[asset].maxRedeemPerBlock;
        tokenConfig[asset].maxRedeemPerBlock = _maxRedeemPerBlock;
        emit MaxRedeemPerBlockChanged(oldMaxRedeemPerBlock, _maxRedeemPerBlock, asset);
    }
```
As noted in the functions above from the UStbMinting contract tokenConfig is not validated for activeness before setting max mint and max redeem per block
##### **Recommendation:**
```solidity
    function _setMaxMintPerBlock(uint128 _maxMintPerBlock, address asset) internal {
+++    if (!tokenConfig[asset].isActive) revert UnsupportedAsset();
        uint128 oldMaxMintPerBlock = tokenConfig[asset].maxMintPerBlock;
        tokenConfig[asset].maxMintPerBlock = _maxMintPerBlock;
        emit MaxMintPerBlockChanged(oldMaxMintPerBlock, _maxMintPerBlock, asset);
    }
...
    function _setMaxRedeemPerBlock(uint128 _maxRedeemPerBlock, address asset) internal {
+++    if (!tokenConfig[asset].isActive) revert UnsupportedAsset();
        uint128 oldMaxRedeemPerBlock = tokenConfig[asset].maxRedeemPerBlock;
        tokenConfig[asset].maxRedeemPerBlock = _maxRedeemPerBlock;
        emit MaxRedeemPerBlockChanged(oldMaxRedeemPerBlock, _maxRedeemPerBlock, asset);
    }
```
...
## [QA-05] Whitelisted Address DOS Before Token Transfer
##### **Issue Description:**
https://github.com/code-423n4/2024-11-ethena-labs/blob/main/contracts/ustb/UStb.sol#L202-L209
```solidity
  function _beforeTokenTransfer(address from, address to, uint256) internal virtual override {
        // State 2 - Transfers fully enabled except for blacklisted addresses
        if (transferState == TransferState.FULLY_ENABLED) {
            ...
            // State 1 - Transfers only enabled between whitelisted addresses
        } else if (transferState == TransferState.WHITELIST_ENABLED) {
            ...
                // whitelisted user can burn
            } else if (
                hasRole(WHITELISTED_ROLE, msg.sender) &&
                hasRole(WHITELISTED_ROLE, from) &&
                hasRole(WHITELISTED_ROLE, to) &&
                !hasRole(BLACKLISTED_ROLE, msg.sender) &&
                !hasRole(BLACKLISTED_ROLE, from) &&
                !hasRole(BLACKLISTED_ROLE, to)
            ) {
>>>                // n.b. an address can be whitelisted and blacklisted at the same time
                // normal case
            } else {
                revert OperationNotAllowed();
            }
            // State 0 - Fully disabled transfers
        } else if (transferState == TransferState.FULLY_DISABLED) {
            ...
        }
    }
```
Protocol comment description in the code above notes that an address can be whitelisted and blacklisted at the same time but that is not the case as code would revert when an address is whitelisted and blacklisted as the condition would return false.
it is also noted in documentation that when - WHITELIST_ENABLED: Only whitelisted addresses can send and receive this token. which shows a whitelisted address should be allowed to carry out normal activities even if they are blacklisted
##### **Recommendation:**
Protocol should adjust code to ensure whitelisted address can make normal calls and implementation before token transfer 

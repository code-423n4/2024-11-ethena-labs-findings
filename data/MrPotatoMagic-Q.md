# Quality Assurance

## [L-01] stablesDeltaLimit allows upto 1e18 worth of difference which can lead to UStb peg stability issues

If we view Line 574, we can see it calculates the `differenceInBPS`. If we assume difference as 1e18, `STABLES_RATIO_MULTIPLIER` (which is 10000) and `ustbAmount` as 1e18, it would set `differenceInBPS` to 10000, which on Line 577 would evaluate to true when differenceInBPS is checked to be less than equal to stablesDeltaLimit.

In short, this means that for 0 amount of collateral provided, 1 UStb can be minted. This breaks the UStb peg. Although this is all admin controlled, the behaviour should still be noted and fixed possibly to disallow any discrepancies (assuming admin does their work correctly).
```
File: UStbMinting.sol
565:         normalizedCollateralAmount = ustbDecimals > collateralDecimals
566:             ? collateralAmount * scale
567:             : collateralAmount / scale;
568: 
569:         uint128 difference = normalizedCollateralAmount > ustbAmount
570:             ? normalizedCollateralAmount - ustbAmount
571:             : ustbAmount - normalizedCollateralAmount;
572: 
573:            
574:         uint128 differenceInBps = (difference * STABLES_RATIO_MULTIPLIER) / ustbAmount;
575: 
576:         if (orderType == OrderType.MINT) {
577:             return ustbAmount > normalizedCollateralAmount ? differenceInBps <= stablesDeltaLimit : true;
578:         } else {
579:             return normalizedCollateralAmount > ustbAmount ? differenceInBps <= stablesDeltaLimit : true;
580:         }
```

## [L-02] setApprovedBeneficiary does not check if msg.sender is a whitelisted benefactor

The function does not check if msg.sender is a whitelisted benefactor. While this does not seem to pose any risk at the moment, a check should be added to ensure this is only accessible to those whitelisted.
```
File: UStbMinting.sol
416:     function setApprovedBeneficiary(address beneficiary, bool status) public {
417:         if (status) {
418:             if (!_approvedBeneficiariesPerBenefactor[msg.sender].add(beneficiary)) {
419:                 revert InvalidBeneficiaryAddress();
420:             } else {
421:                 emit BeneficiaryAdded(msg.sender, beneficiary);
422:             }
423:         } else {
424:             if (!_approvedBeneficiariesPerBenefactor[msg.sender].remove(beneficiary)) {
425:                 revert InvalidBeneficiaryAddress();
426:             } else {
427:                 emit BeneficiaryRemoved(msg.sender, beneficiary);
428:             }
429:         }
430:     }
```

## [L-03] Redeem does not support functionality for an approved beneficiary to redeem tokens to a benefactor

During minting, we know it is possible for the benefactor or delegated signer to mint UStb to a different beneficiary address.

During redeeming though, it always attempts to burn tokens from the `benefactor`. This means it expects the benefactor to hold the UStb. The issue is that during a mint when tokens are sent to an approved beneficiary, it could expect that approved beneficiary of the benefactor to be a whitelisted benefactor as well. This means the Ethena team is forced to make the approved beneficiaries of a benefactor as whitelisted benefactors as well for the redeem to work. While an approved beneficiary can transfer the tokens to the benefactor, that should not be considered as a valid way out of the issue since approved beneficiaries can even be a contract. 

Since benefactors are entities that need to undergo KYC by Ethena (as per [README](https://github.com/code-423n4/2024-11-ethena-labs/tree/main#5-benefactor)), this could be troublesome when the beneficiary is a contract, which cannot undergo a KYC. I'm assuming this could pose legal risks if the benefactor and contract become malicious. The issue is worth noting and possible implementing a way for approved beneficiaries to redeem for a benefactor would be good.

```
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
        ...

        ustb.burnFrom(order.benefactor, order.ustb_amount);
        _transferToBeneficiary(order.beneficiary, order.collateral_asset, order.collateral_amount);
        ...
    }
```

## [L-04] Missing ReentrancyGuard constructor call in UStbMinting.sol

The constructor in the minting contract does not call the inherited reentrancy guard contact's constructor, which initializes the status to NOT_ENTERED.
```solidity
File: UStbMinting.sol
160:     constructor(
161:         address[] memory _assets,
162:         TokenConfig[] memory _tokenConfig,
163:         GlobalConfig memory _globalConfig,
164:         address[] memory _custodians,
165:         address _admin
166:     ) {
```

## [L-05] Mint and redeem limits can be DOSed

Mint and redeem block limits can be DOSed always by minting and redeeming back to back in a block. This can be done only when a user's transaction is spotted in the mempool and frontrunning it. While this can be managed by removing the malicious minter/redeemer or increasing the minting and redeeming limit, it should be noted.

```solidity
modifier belowMaxMintPerBlock(uint128 mintAmount, address asset) {
        TokenConfig memory _config = tokenConfig[asset];
        if (!_config.isActive) revert UnsupportedAsset();
        if (totalPerBlockPerAsset[block.number][asset].mintedPerBlock + mintAmount > _config.maxMintPerBlock) {
            revert MaxMintPerBlockExceeded();
        }
        _;
    }
```
Real Pastel Starfish

high

# Lenders can't withdraw all claimable collateral (ERC721) while auctions ongoing
## Summary

In ERC721 pools, when all NFTs are initially holded by the borrowers (still no claimable collateral), then some of them get kicked and `bucketTake` is called, there's a big chance that the lenders won't be able to withdraw the claimable collateral liquidated until some of the auctions finishes. 

## Vulnerability Detail

In ERC721 pools, the NFTs used as collateral that the contract holds are stored in two different arrays depending on the state of the collateral: `borrowerTokenIds` and `bucketTokenIds`.

The array `borrowerTokenIds` holds the NFTs that belong to a specific borrower while the array `bucketTokenIds` holds the NFTs that the lenders can claim. When a `take` (or `bucketTake`) action is called, the `_rebalanceTokens` function is called to transfer the NFTs sold in that `take` action from the borrower's array to the lender's array.

Because of the nature of non-fungible tokens, is not possible to transfer 0.5 NFTs so the `_rebalanceTokens` will only transfer NFTs from the borrower to the lenders when the borrower loses the full unit of collateral. That means that if a borrower has `collateral = 0.1e18`, his `borrowerTokenIds` will have a full NFT in it. 

Here is the function `rebalanceTokens`:
https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/ERC721Pool.sol#L525-L547

```solidity
function _rebalanceTokens(
    address borrowerAddress_,
    uint256 borrowerCollateral_
) internal {
    // rebalance borrower's collateral, transfer difference to floor collateral from borrower to pool claimable array
    uint256[] storage borrowerTokens = borrowerTokenIds[borrowerAddress_];

    uint256 noOfTokensPledged    = borrowerTokens.length;
    /*
        eg1. borrowerCollateral_ = 4.1, noOfTokensPledged = 6; noOfTokensToTransfer = 1
        eg2. borrowerCollateral_ = 4, noOfTokensPledged = 6; noOfTokensToTransfer = 2
    */
    uint256 borrowerCollateralRoundedUp = (borrowerCollateral_ + 1e18 - 1) / 1e18;
    uint256 noOfTokensToTransfer = noOfTokensPledged - borrowerCollateralRoundedUp;

    for (uint256 i = 0; i < noOfTokensToTransfer;) {
        uint256 tokenId = borrowerTokens[--noOfTokensPledged]; // start with moving the last token pledged by borrower
        borrowerTokens.pop();                                  // remove token id from borrower
        bucketTokenIds.push(tokenId);                          // add token id to pool claimable tokens

        unchecked { ++i; }
    }
}
```

In cases when `bucketTake` is called, is possible that only a partial amount of a unit of collateral is taken due to the constraint in that bucket's deposits. That means that maybe only `0.5e18` units of collateral is taken instead of the full unit (`1e18`). In these cases when only a partial unit of collateral is taken, no NFTs will be transferred from the borrower's array to the lender's array. 

Because the collateral units are only moved from `borrowerTokenIds` to `bucketTokenIds` when a full unit of collateral is taken, this allows a scenario where a lender wants to withdraw claimable collateral from a bucket and is not possible because the  `bucketTokenIds` doesn't have enough NFTs. 

## Impact

If the conditions mentioned above are met, there will be a situation where a lender can't withdraw his NFT until some auction finished, so that could be a maximum of 72 hours of locked assets. 

There is no low-probability prerequisites and the impact is a freezing of funds, so setting the severity to be high.

## Proof of Concept

Imagine this scenario where two borrowers are in an ERC721 pool:


 - borrower1: collateral = 1e18
 - borrower2: collateral = 1e18
 
 - pool.borrowerTokenIds[borrower1].length = 1
 - pool.borrowerTokenIds[borrower2].length = 1
 - pool.bucketTokenIds.length = 0

Both borrowers are kicked. When some time has passed, `bucketTake` is called on both liquidating `0.5e18` collateral on both borrowers. 

Because both borrowers still have `0.5e18` collateral, no NFTs are transfered from `borrowerTokenIds` to `bucketTokenIds`. 

Now, there's a total of `1e18` claimable collateral in the buckets so if a lender merges the collateral in the same bucket and wants to redeem LPB in order to withdraw the full NFT it should be possible but it isn't. 

When the lender tries to withdraw the NFT, the call will revert because the `bucketTokenIds` array doesn't have NFTs in it. That NFT will be locked in the contract untill one of the auctions settles. 

Here is a coded POC of this scenario:

Paste this code in a new file inside `/tests/forge/unit/ERC721Pool` and run it with `forge test --match-test testStuckNFTs -vvv`
```solidity
// SPDX-License-Identifier: UNLICENSED
pragma solidity 0.8.18;

import { ERC721HelperContract } from './ERC721DSTestPlus.sol';

import 'src/libraries/helpers/PoolHelper.sol';

contract ERC721PoolLiquidationsDepositTakeTest is ERC721HelperContract {

    address internal _borrower;
    address internal _borrower2;
    address internal _lender;
    address internal _taker;

    function testStuckNFTs() external {
        _startTest();

        _borrower  = makeAddr("borrower");
        _borrower2 = makeAddr("borrower2");
        _lender    = makeAddr("lender");
        _taker     = makeAddr("taker");

        // deploy pool
        uint256[] memory subsetTokenIds = new uint256[](0);
        _pool = _deploySubsetPool(subsetTokenIds);

        _mintAndApproveQuoteTokens(_lender,  10 * 1e18);
        _mintAndApproveQuoteTokens(_borrower, 10 * 1e18);

        _mintAndApproveCollateralTokens(_borrower,  5);
        _mintAndApproveCollateralTokens(_borrower2, 5);

        // Lender adds quote token at 3 buckets
        _addInitialLiquidity({
            from:   _lender,
            amount: 5 * 1e18,
            index:  4156 // price of 1
        });
        _addInitialLiquidity({
            from:   _lender,
            amount: 0.5 * 1e18,
            index:  4157 // price of 1.005
        });
        _addInitialLiquidity({
            from:   _lender,
            amount: 0.5 * 1e18,
            index:  4158 // price of 1.010025
        });

        // first borrower adds collateral token and borrows
        uint256[] memory tokenIdsToAdd = new uint256[](1);
        tokenIdsToAdd[0] = 1;
        uint256 expectedNewLup = 1 * 1e18;

        // Both borrowers deposit 1 NFT as collateral and borrow 0.9e18 quote tokens
        _pledgeCollateral({
            from:     _borrower,
            borrower: _borrower,
            tokenIds: tokenIdsToAdd
        });
        _borrow({
            from:       _borrower,
            amount:     0.9 * 1e18,
            indexLimit: 4156,
            newLup:     expectedNewLup
        });

        tokenIdsToAdd = new uint256[](1);
        tokenIdsToAdd[0] = 8;
        _pledgeCollateral({
            from:     _borrower2,
            borrower: _borrower2,
            tokenIds: tokenIdsToAdd
        });
        _borrow({
            from:       _borrower2,
            amount:     0.9 * 1e18,
            indexLimit: 4156,
            newLup:     expectedNewLup
        });

        /*****************************/
        /*** Assert pre-kick state ***/
        /*****************************/

        _assertPool(
            PoolParams({
                htp:                  0.900865384615384616 * 1e18,
                lup:                  expectedNewLup,
                poolSize:             6 * 1e18,
                pledgedCollateral:    2 * 1e18,
                encumberedCollateral: 1.801730769230769232 * 1e18,
                poolDebt:             1.801730769230769232 * 1e18,
                actualUtilization:    0,
                targetUtilization:    1 * 1e18,
                minDebtAmount:        0.090086538461538462 * 1e18,
                loans:                2,
                maxBorrower:          address(_borrower),
                interestRate:         0.05 * 1e18,
                interestRateUpdate:   _startTime
            })
        );
        _assertBorrower({
            borrower:                  _borrower,
            borrowerDebt:              0.900865384615384616 * 1e18,
            borrowerCollateral:        1 * 1e18,
            borrowert0Np:              1.037619811928824661 * 1e18,
            borrowerCollateralization: 1.110043761340591311 * 1e18
        });
        _assertBorrower({
            borrower:                  _borrower2,
            borrowerDebt:              0.900865384615384616 * 1e18,
            borrowerCollateral:        1 * 1e18,
            borrowert0Np:              1.037619811928824661 * 1e18,
            borrowerCollateralization: 1.110043761340591311 * 1e18
        });

        assertEq(_quote.balanceOf(_lender), 4 * 1e18);

        // Skip to make both borrowers undercollateralized
        skip(1000 days);

        _kick({
            from:           _lender,
            borrower:       _borrower,
            debt:           1.033123628629169037 * 1e18,
            collateral:     1 * 1e18,
            bond:           0.015683167828397025 * 1e18,
            transferAmount: 0.015683167828397025 * 1e18
        });

        _kick({
            from:           _lender,
            borrower:       _borrower2,
            debt:           1.033123628629169037 * 1e18,
            collateral:     1 * 1e18,
            bond:           0.015683167828397025 * 1e18,
            transferAmount: 0.015683167828397025 * 1e18
        });

        /******************************/
        /*** Assert Post-kick state ***/
        /******************************/

        _assertPool(
            PoolParams({
                htp:                  0,
                lup:                  1e18,
                poolSize:             6.224839014823433514 * 1e18,
                pledgedCollateral:    2 * 1e18,
                encumberedCollateral: 2.066247257258338074 * 1e18,
                poolDebt:             2.066247257258338074 * 1e18,
                actualUtilization:    0.300288461538461539 * 1e18,
                targetUtilization:    0.900865384615384616 * 1e18,
                minDebtAmount:        0,
                loans:                0,
                maxBorrower:          address(0),
                interestRate:         0.045 * 1e18,
                interestRateUpdate:   block.timestamp
            })
        );
        _assertBorrower({
            borrower:                  _borrower,
            borrowerDebt:              1.033123628629169037 * 1e18,
            borrowerCollateral:        1e18,
            borrowert0Np:              1.037619811928824661 * 1e18,
            borrowerCollateralization: 0.967938368931586519 * 1e18
        });
        _assertBorrower({
            borrower:                  _borrower2,
            borrowerDebt:              1.033123628629169037 * 1e18,
            borrowerCollateral:        1e18,
            borrowert0Np:              1.037619811928824661 * 1e18,
            borrowerCollateralization: 0.967938368931586519 * 1e18
        });

        skip(7 hours);

        _assertAuction(
            AuctionParams({
                borrower:          _borrower,
                active:            true,
                kicker:            _lender,
                bondSize:          0.015683167828397025 * 1e18,
                bondFactor:        0.015180339887498948 * 1e18,
                kickTime:          block.timestamp - 7 hours,
                referencePrice:    1.189955306913139289 * 1e18,
                totalBondEscrowed: 0.03136633565679405 * 1e18,
                auctionPrice:      0.841425466827200184 * 1e18,
                debtInAuction:     2.066247257258338074 * 1e18,
                thresholdPrice:    1.033160779290608796 * 1e18,
                neutralPrice:      1.189955306913139289 * 1e18
            })
        );

        // Make deposit take with both borrowers using bucket of index 4157 and 4158
        _depositTake({
            from:             _taker,
            borrower:         _borrower,
            kicker:           _lender,
            index:            4157,
            collateralArbed:  0.529371703955175008 * 1e18,
            quoteTokenAmount: 0.526738013885746281 * 1e18,
            bondChange:       0.007996062082451769 * 1e18,
            isReward:         true,
            lpAwardTaker:     0,
            lpAwardKicker:    0.007707167363903366 * 1e18
        });

        _depositTake({
            from:             _taker,
            borrower:         _borrower2,
            kicker:           _lender,
            index:            4158,
            collateralArbed:  0.532018562474950879 * 1e18,
            quoteTokenAmount: 0.526738013885746281 * 1e18,
            bondChange:       0.007996062082451769 * 1e18,
            isReward:         true,
            lpAwardTaker:     0,
            lpAwardKicker:    0.007707167363903366 * 1e18
        });

        _assertBorrower({
            borrower:                  _borrower,
            borrowerDebt:              0.514418827487314285 * 1e18,
            borrowerCollateral:        0.470628296044824992 * 1e18,
            borrowert0Np:              1.097764454523452827 * 1e18,
            borrowerCollateralization: 0.914873777741797019 * 1e18
        });

        _assertBorrower({
            borrower:                  _borrower2,
            borrowerDebt:              0.514418827487314285 * 1e18,
            borrowerCollateral:        0.467981437525049121 * 1e18,
            borrowert0Np:              1.103973305914074461 * 1e18,
            borrowerCollateralization: 0.909728440171816372 * 1e18
        });

        // Lender merges collateral from index 4157 to 4158
        uint256[] memory removalIndexes = new uint256[](1);
        removalIndexes[0] = 4157;
        _mergeOrRemoveCollateral({
            from:                    _lender,
            toIndex:                 4158,
            noOfNFTsToRemove:        2,
            collateralMerged:        0.529371703955175008 * 1e18,
            removeCollateralAtIndex: removalIndexes,
            toIndexLps:              0.505181261058610317 * 1e18
        });

        // Now, lender has at least 1 NFT in bucket 4158
        _assertBucket({
            index:        4158,
            lpBalance:    1.012888428422513683 * 1e18,
            collateral:   1.061390266430125887 * 1e18,
            deposit:      1,
            exchangeRate: 1.037483903606589028 * 1e18
        });

        // When lender tries to withdraw that collateral, call reverts with EvmError: Revert
        // This is because `_removeCollateral` tries first to call bucketTokenIds(0) and this call will revert because array is empty
        _removeCollateral({
            from:     _lender,
            amount:   1,
            index:    4158,
            lpRedeem: 1 // This arg doesn't matter because `bucketTokenIds(0)` call reverts first
        });
    }
}

```

As the test is proving, the lender owns `1.06e18` units of collateral in bucket 4158 so he should be allowed to withdraw a full NFT,  but the transaction is reverted because there's no NFTs in the `bucketTokenIds` array.

## Code Snippet

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/ERC721Pool.sol#L525-L547

## Tool used

Manual Review and reading Hyh's reports :)

## Recommendation

Consider changing `rebalanceTokens` so it rounds borrower's collateral to `floor` instead of `ceil`. So if a borrower has `collateral = 0.9e18`, move the NFT from borrower's array to lender's array. 


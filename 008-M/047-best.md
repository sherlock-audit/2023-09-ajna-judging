Perfect Vinyl Whale

high

# Users could make Lup goes below HTP by using LenderActions::moveQuoteToken in a specific case
## Summary
Moving deposits from lup index bucket ( index == LUP) to smaller index could bypass the invariant (LUP >= HTP) check
## Vulnerability Detail
In moveQuoteToken(), after removing deposits from the original bucket and adding deposits to the destination bucket, the code will check if the final result pushes LUP below HTP. If yes then the tx will revert.
https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/LenderActions.sol#L322-L333

When fromIndex == Lup index, moving deposits to lower bucket can actually make LUP goes down. While htp is unchanged (based on loans only), there will be chances that LUP goes below HTP.

The problem is  when fromIndex == Lup index , the code skips checking if lup goes below htp due to the first condition of the invariant check (params_.fromIndex < params_.toIndex) 



Consider this example: 
Bucket #3: 10 QTs
Bucket #2: 15 QTs
Bucket #1: 5 QTs

Debt: 20QTs

As definition,

_We can think of a
bucket’s deposit as being utilized if the sum of all deposits in buckets priced higher than it is less
than the total debt of all borrowers in the pool. The lowest price among utilized buckets or
“lowest utilized price” is called the LUP_

so the original LUP index is 2.

Now an user moves 10 QT from bucket #2  (LUP index) to bucket #1 (lower index). The new amount of QTs in each bucket will be:

Bucket #3: 10 QTs
Bucket #2: 5 QTs
Bucket #1: ~15QTs 

Debt: 20 QTs

Now the new LUP index is 1. If the HTP is somewhere between 2 and 1, moving deposits will make LUP < HTP, yet it still bypasses the invariant check.
## Impact
Breaking one of the most important invariants of the protocol.
As stated in the whitepaper,

_In other words: in order to remove a deposit, the user must ensure that this action does not cause the LUP to move below the “highest threshold price”, HTP, which is the threshold price of the least collateralized loan_

## Code Snippet
https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/LenderActions.sol#L322-L333
## Tool used

Manual Review

## Recommendation
Consider either  adding this case (from index = lup index) or removing the first condition ( params_.fromIndex < params_.toIndex) of the invariant check.
```solidity
        if (
            //params_.fromIndex < params_.toIndex
            //&&
            (
                // check loan book's htp doesn't exceed new lup
                vars.htp > lup_
                ||
                // ensure that pool debt < deposits after move
                // this can happen if deposit fee is applied when moving amount
                (poolState_.debt != 0 && poolState_.debt > Deposits.treeSum(deposits_))
            )
        ) revert LUPBelowHTP();
```
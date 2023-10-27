Bumpy Punch Albatross

medium

# HPB may be incorrectly bankrupt due to use of unscaled value in `_forgiveBadDebt`
## Summary

An unscaled value is used in place of where a scaled value should be used in the bankruptcy check in  `_forgiveBadDebt`. This may cause the bucket to be incorrectly marked as bankrupt, losing user funds.

## Vulnerability Detail

At the end of `_forgiveBadDebt`, we do a usual bankruptcy check in which we check whether the remaining deposit and collateral will be little enough that the exchange rate will round to 0, in which case we mark the bucket as bankrupt, setting the bucket lps and effectively all user lps as 0.

The problem lies in the fact that we use `depositRemaining` as part of this check, which represents an unscaled value. As a result, when computing whether the exchange rate rounds to 0 our logic is off by a factor of the bucket's scale. The bucket may be incorrectly marked as bankrupt if the unscaled `depositRemaining` would result in an exchange rate of 0 when the scaled `depositRemaining` would not.

## Impact

Loss of user funds.

## Code Snippet

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L485
```solidity
// If the remaining deposit and resulting bucket collateral is so small that the exchange rate
// rounds to 0, then bankrupt the bucket.  Note that lhs are WADs, so the
// quantity is naturally 1e18 times larger than the actual product
// @audit depositRemaining should be a scaled value
if (depositRemaining * Maths.WAD + hpbBucket.collateral * _priceAt(index) <= bucketLP) {
    // existing LP for the bucket shall become unclaimable
    hpbBucket.lps            = 0;
    hpbBucket.bankruptcyTime = block.timestamp;

    emit BucketBankruptcy(
        index,
        bucketLP
    );
}
```

## Tool used

Manual Review

## Recommendation

Simply scale `depositRemaining` before doing the bankruptcy check, e.g. something like:

```solidity
depositRemaining = Maths.wmul(depositRemaining, scale);
if (depositRemaining * Maths.WAD + hpbBucket.collateral * _priceAt(index) <= bucketLP) {
    // existing LP for the bucket shall become unclaimable
    hpbBucket.lps            = 0;
    hpbBucket.bankruptcyTime = block.timestamp;

    emit BucketBankruptcy(
        index,
        bucketLP
    );
}
```
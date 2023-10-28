Restless White Bee

medium

# BucketTake rewards can go to an insolvent bucket
## Summary

`_rewardBucketTake()` adds bucket take rewards in a form of bucket LP, but doesn't check for bucket bankruptcy.

## Vulnerability Detail

As all the subsequent operations deal only with LPs added later than bankruptcy time it doesn't make sense to add rewards to a bucket at the moment of its bankruptcy, they will be lost for beneficiaries.

## Impact

If `bankruptcyTime == block.timestamp` during `bucketTake()` then all the rewards will be lost for beneficiaries, i.e. both taker and kicker will not be able to withdraw anything.

## Code Snippet

`_rewardBucketTake()` doesn't check the bucket for being bankrupt at the current block (and `bankruptcyTime` isn't read and checked before that):

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/TakerActions.sol#L607-L651

```solidity
    function _rewardBucketTake(
        ...
    ) internal {
        Bucket storage bucket = buckets_[bucketIndex_];

>>      uint256 bankruptcyTime = bucket.bankruptcyTime;
        uint256 scaledDeposit  = Maths.wmul(vars.unscaledDeposit, vars.bucketScale);
        uint256 totalLPReward;
        uint256 takerLPReward;
        uint256 kickerLPReward;

        // if arb take - taker is awarded collateral * (bucket price - auction price) worth (in quote token terms) units of LPB in the bucket
        if (!depositTake_) {
            takerLPReward = Buckets.quoteTokensToLP(
                bucket.collateral,
                bucket.lps,
                scaledDeposit,
                Maths.wmul(vars.collateralAmount, vars.bucketPrice - vars.auctionPrice),
                vars.bucketPrice,
                Math.Rounding.Down
            );
            totalLPReward = takerLPReward;

>>          Buckets.addLenderLP(bucket, bankruptcyTime, msg.sender, takerLPReward);
        }

        // the bondholder/kicker is awarded bond change worth of LPB in the bucket
        if (vars.isRewarded) {
            kickerLPReward = Buckets.quoteTokensToLP(
                bucket.collateral,
                bucket.lps,
                scaledDeposit,
                vars.bondChange,
                vars.bucketPrice,
                Math.Rounding.Down
            );
            totalLPReward  += kickerLPReward;

>>          Buckets.addLenderLP(bucket, bankruptcyTime, vars.kicker, kickerLPReward);
```

So, if `bankruptcyTime == block.timestamp` at the time of `bucketTake()` then `lender.depositTime` will be set to `bucket.bankruptcyTime`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/internal/Buckets.sol#L77-L91

```solidity
    function addLenderLP(
        ...
    ) internal {
        if (lpAmount_ != 0) {
            Lender storage lender = bucket_.lenders[lender_];

            if (bankruptcyTime_ >= lender.depositTime) lender.lps = lpAmount_;
            else lender.lps += lpAmount_;

>>          lender.depositTime = block.timestamp;
        }
    }
```

And both taker and kicker will not be able to withdraw any funds thereafter as their LP balance will be deemed empty:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/LenderActions.sol#L516-L520

```solidity
        Lender storage lender = bucket.lenders[msg.sender];

        uint256 lenderLpBalance;
>>      if (bucket.bankruptcyTime < lender.depositTime) lenderLpBalance = lender.lps;
        if (lenderLpBalance == 0 || lpAmount_ > lenderLpBalance) revert InsufficientLP();
```

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/LenderActions.sol#L411-L415

```solidity
        uint256 depositTime = lender.depositTime;

        RemoveDepositParams memory removeParams;

>>      if (bucket.bankruptcyTime < depositTime) removeParams.lpConstraint = lender.lps;
```

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/LenderActions.sol#L271-L275

```solidity
        vars.fromBucketDepositTime = fromBucketLender.depositTime;

        vars.toBucketPrice         = _priceAt(params_.toIndex);

>>      if (fromBucket.bankruptcyTime < vars.fromBucketDepositTime) vars.fromBucketLenderLP = fromBucketLender.lps;
```

## Tool used

Manual Review

## Recommendation

For uniformity consider adding the similar check to `_rewardBucketTake()`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/TakerActions.sol#L607-L618

```diff
    function _rewardBucketTake(
        ...
    ) internal {
        Bucket storage bucket = buckets_[bucketIndex_];

        uint256 bankruptcyTime = bucket.bankruptcyTime;
+       
+       // cannot deposit bucket take rewards in the same block when bucket becomes insolvent
+       if (bankruptcyTime_ == block.timestamp) revert BucketBankruptcyBlock();
```
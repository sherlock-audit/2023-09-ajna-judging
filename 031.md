Real Pastel Starfish

medium

# Loss of user's LPB in `removeQuoteToken` due to lack of rounding
## Summary

The function `removeQuoteToken` receives `maxAmount_` as an argument, that is the amount (in WAD) of quote token to be removed by the lender. Because that value is not rounded to quote token's precison, a user may burn more `LPB` than necessary in order to withdraw the same amount of quote token. 

## Vulnerability Detail

The `removeQuoteToken` function is used by the lenders to withdraw deposits (quote tokens) from a bucket. The user must specify the maximum amount of quote token that must be withdrawn from that bucket via the `maxAmount_` argument. Then, the function calculates the actual tokens that have been withdrawn and that value is set in `removedAmount_`. The last step of the function is send `removedAmount_ / QUOTE_SCALE` tokens to the user. 

The variable `removedAmount_` has 18 decimals of precision and is not rounded until the moment of sending funds to the user, so the function is using the non-rounded value to calculate the amount of `LPB` to be burned. 

This means that we are taking a greater number to calculate the amount of `LPB` burned (`removedAmount_` without rounding) but we're sending the user an amount of tokens as if `removedAmount` was rounded. 

Imagine this scenario, we have a pool with `USDC` as quote token (6 decimals) and 2 lenders want to withdraw some deposits from buckets: 

```text
Lender1 and lender2 remove quote token from buckets with exchangeRate of 1:1:
(for the example we assume that the lenders and buckets have enough LPB to support the withdrawal)

Lender1:
    - maxAmount_ = 1000000999999999999 (1.000000999999999999 in WAD)
    - removedAmount_ = maxAmount_
    - redeemedLP = maxAmount_
    - quoteTokenReceived = maxAmount_ / 1e12 = 1e6
    
Lender2:
    - maxAmount_ = 1e18 (1 in WAD)
    - removedAmount = maxAmount_
    - redeemedLP = maxAmount_
    - quoteTokenReceived = maxAmount_ / 1e12 = 1e6
    

Lender1 has burned 1000000999999999999 LPB.
Lender2 has burned 1000000000000000000 LPB.
Both lenders have received the same amount of quote tokens (1e6).
```

As showed in the example, some lenders may burn more or less `LPB` while receiving the same amount of quote tokens, thus violating the system design.

## Impact

Everytime users call `removeQuoteToken` without rounding the `maxAmount_` variable to quote token's precision they'll be burning excess `LPB`.

In pools where quote token's precision is lower than 18 decimals, there is no low-probability prerequisites and the impact will be the loss of assets (LPB) from the users, as well as an unfairly advantage from the users that know this issue and how to avoid it. That's why severity is set to medium. 

## Code Snippet

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/base/Pool.sol#L225-L262

## Tool used

Manual Review

## Recommendation

Round the argument `maxAmount_` to quote token's precision to avoid the excess of `LPB` burned. 

```diff
function removeQuoteToken(
    uint256 maxAmount_,
    uint256 index_
) external override nonReentrant returns (uint256 removedAmount_, uint256 redeemedLP_) {
    _revertIfAuctionClearable(auctions, loans);

    PoolState memory poolState = _accruePoolInterest();
    
+   // round to token precision
+   maxAmount_ = _roundToScale(maxAmount_, poolState.quoteTokenScale);

    // ...
}
```
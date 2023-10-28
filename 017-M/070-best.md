Restless White Bee

medium

# Adding yearly interest overcharges borrowers when pool rate is high due to market expectations
## Summary

Mechanics of adding square root of pool's rate directly to `npTpRatio` doesn't fit in the situation when pool's rate is determined by the current market expectation instead of long-term volatility.

## Vulnerability Detail

While it's true that equilibrium rate shows the expected price compared to the market one, it is not true that this price is simply current estimate of the market price with yearly interest added to it: the time when market expects some particular price movement is not observable (e.g. is it next week or within one year), so assumption of perpentual interest rate difference being the only factor driving equilibrium rate can't be universally applicable to all the assets.

In other words the fact that market participants are ready to pay say `100%` annualized interest cannot be deemed equal to that expected price is `2x` of the current market one. For example, suppose current consensys is that price will raise by `1.5x` in a half of a year and then stay put (i.e. there are some publicly known forces that form a ceiling at some level). Borrowers are willing to pay up to `100%` interest for half of a year as that's `50%`, which is what the expected profit is. Expected market price in the same time isn't `2x` of the current and such an estimate allows kickers to earn more rewards due to elevated NP.

## Impact

Kicker reward can be substantially overstated at the expense of the borrower.

## Code Snippet

When pool rate is high the `npTpRatio` will be high:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/internal/Loans.sol#L102-L105

```solidity
        // update Np to Tp ratio of borrower
        if (npTpRatioUpdate_) {
            borrower_.npTpRatio = 1.04 * 1e18 + uint256(PRBMathSD59x18.sqrt(int256(poolRate_))) / 2;
        }
```

Say for `200%` rate it will be `npTpRatio = (1.04 + math.sqrt(2) / 2) = 1.7471`.

So `neutralPrice` will also be high, it can be said that it will almost always have significant margin above the market:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/KickerActions.sol#L307-L313

```solidity
        // calculate auction params
        // neutral price = Tp * Np to Tp ratio
        // neutral price is capped at 50 * max pool price
        vars.neutralPrice = Maths.min(
>>          Math.mulDiv(vars.borrowerDebt, borrower.npTpRatio, vars.borrowerCollateral),
            MAX_INFLATED_PRICE
        );
```

I.e. `NP = 1.7471 * TP`, while, for example, at 150% collaterization, typical for `ETH` and `BTC` markets that can be deemed averagely volatile (in the sense that there are less volatile stable coin markets and more volatile NFT markets), it will be `NP = 1.7471 * TP = 1.7471 * market_price / 1.5 = 1.16 * market_price`. With such a margin and without significant market movements the auction will be settled below that.

`bondFactor_` will be maximized at `3%`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/helpers/PoolHelper.sol#L418-L426

```solidity
    function _bondParams(
        uint256 borrowerDebt_,
        uint256 npTpRatio_
    ) pure returns (uint256 bondFactor_, uint256 bondSize_) {
        // bondFactor = min((NP-to-TP-ratio - 1)/10, 0.03)
>>      bondFactor_ = Maths.min(
>>          0.03 * 1e18,
>>          (npTpRatio_ - 1e18) / 10
        );
```

So kicker's reward will be placed somewhere at `(0, 1) * bondFactor_` as due to `neutralPrice_` being big, `auctionPrice_ = market price` will almost always be between `neutralPrice_`, which will be above the market due to elevation, and `thresholdPrice`, which were initially below the market (barring abrupt movements):

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/helpers/PoolHelper.sol#L382-L411

```solidity
    function _bpf(
        ...
    ) pure returns (int256) {
        int256 thresholdPrice = int256(Maths.wdiv(debt_, collateral_));

        int256 sign;
        if (thresholdPrice < int256(neutralPrice_)) {
            // BPF = BondFactor * min(1, max(-1, (neutralPrice - price) / (neutralPrice - thresholdPrice)))
            sign = Maths.minInt(
                1e18,
                Maths.maxInt(
                    -1 * 1e18,
                    PRBMathSD59x18.div(
>>                      int256(neutralPrice_) - int256(auctionPrice_),
>>                      int256(neutralPrice_) - thresholdPrice
                    )
                )
            );
        } else {
            int256 val = int256(neutralPrice_) - int256(auctionPrice_);
            if (val < 0 )      sign = -1e18;
            else if (val != 0) sign = 1e18;
        }

>>      return PRBMathSD59x18.mul(int256(bondFactor_), sign);
    }
```

This way the expectation of some price advancement will make kicker rewards guaranteed and outsized. Also, there will be a substantial enough portion going to reserves, `(5 * bondFactor - bpf) / 4 - bpf = 5 * (bondFactor - bpf) / 4`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/TakerActions.sol#L732-L741

```solidity
        // price is the current auction price, which is the price paid by the LENDER for collateral
        // from the borrower point of view, there is a take penalty of  (1.25 * bondFactor - 0.25 * bpf)
        // Therefore the price is actually price * (1.0 - 1.25 * bondFactor + 0.25 * bpf)
        uint256 takePenaltyFactor    = uint256(5 * int256(vars.bondFactor) - vars.bpf + 3) / 4;  // Round up
>>      uint256 borrowerPrice        = Maths.floorWmul(vars.auctionPrice, Maths.WAD - takePenaltyFactor);

        // To determine the value of quote token removed from a bucket in a bucket take call, we need to account for whether the bond is
        // rewarded or not.  If the bond is rewarded, we need to remove the bond reward amount from the amount removed, else it's simply the 
        // collateral times auction price.
>>      uint256 netRewardedPrice     = (vars.isRewarded) ? Maths.wmul(Maths.WAD - uint256(vars.bpf), vars.auctionPrice) : vars.auctionPrice;
```

I.e. the pool rate is not only a measure of asset volatility, in which case the approach looks sound, but also a measure of market expectations, in which case it has the side effect of making borrowers pay both an outsized returns to the kickers, outsized contribution to reserves, and also makes kicking risk-free.

## Tool used

Manual Review

## Recommendation

Consider elaborating on the formula to make it more universal for various market drivers.
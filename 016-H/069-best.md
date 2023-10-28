Restless White Bee

high

# Subsequent takes increase next kicker reward, allowing total kicker reward to be artificially amplified by splitting take into a batch
## Summary

Kicker reward is being determined as a proportion of auction price to neutral price (NP) distance to the distance between NP and threshold price (TP). The latter is determined on the fly, with the current debt and collateral, and in the presence of take penalty, actually rises with every take.

This way for any taker it will be profitable to perform many small takes atomically instead of one bigger take, bloating the kicker reward received simply due to rising TP as of time of each subsequent kick.

## Vulnerability Detail

Take penalty being imposed on a borrower worsens its TP. This way a take performed on the big enough debt being auctioned increases the kicker rewards produced by the next take.

This way multiple takes done atomically increase cumulative kicker reward, so if kicker and taker are affiliated then the kicker reward can be augmented above one stated in the protocol logic.

## Impact

Kicker reward can be made excessive at the expense of the reserves part.

## Code Snippet

If `auctionPrice_` is fixed to be `market_price - required_margin` then the bigger `thresholdPrice` the bigger the `sign` and resulting `_bpf()` returned value:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/helpers/PoolHelper.sol#L382-L411

```solidity
    function _bpf(
        ...
    ) pure returns (int256) {
        int256 thresholdPrice = int256(Maths.wdiv(debt_, collateral_));

        int256 sign;
>>      if (thresholdPrice < int256(neutralPrice_)) {
            // BPF = BondFactor * min(1, max(-1, (neutralPrice - price) / (neutralPrice - thresholdPrice)))
            sign = Maths.minInt(
                1e18,
                Maths.maxInt(
                    -1 * 1e18,
                    PRBMathSD59x18.div(
                        int256(neutralPrice_) - int256(auctionPrice_),
>>                      int256(neutralPrice_) - thresholdPrice
                    )
                )
            );
        } else {
            int256 val = int256(neutralPrice_) - int256(auctionPrice_);
            if (val < 0 )      sign = -1e18;
>>          else if (val != 0) sign = 1e18;
        }

        return PRBMathSD59x18.mul(int256(bondFactor_), sign);
    }
```

`thresholdPrice >= int256(neutralPrice_)` will not happen on initial kick as `npTpRatio` is always above 1, but can happen if TP is raised high enough by a series of smaller takes, i.e. taker in some cases can even force kicker reward to be maximal `+1 * bondFactor_`.

`takePenaltyFactor` is paid by the borrower in addition to auction price:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/TakerActions.sol#L724-L736

```solidity
    function _calculateTakeFlowsAndBondChange(
        uint256              totalCollateral_,
        uint256              inflator_,
        uint256              collateralScale_,
        TakeLocalVars memory vars
    ) internal pure returns (
        TakeLocalVars memory
    ) {
        // price is the current auction price, which is the price paid by the LENDER for collateral
        // from the borrower point of view, there is a take penalty of  (1.25 * bondFactor - 0.25 * bpf)
        // Therefore the price is actually price * (1.0 - 1.25 * bondFactor + 0.25 * bpf)
        uint256 takePenaltyFactor    = uint256(5 * int256(vars.bondFactor) - vars.bpf + 3) / 4;  // Round up
        uint256 borrowerPrice        = Maths.floorWmul(vars.auctionPrice, Maths.WAD - takePenaltyFactor);
```

`takePenaltyFactor` can be `3-3.75%` for `vars.bpf > 0` and `bondFactor` maximized to be `3%`, so if TP is above market price by a lower margin, say `TP = 1.025 * market_price`, then the removal of `3-3.75%` part will worsen it off.

The opposite will happen when TP is greater than that, but this calls for the singular executions and doesn't look exploitable.

## Tool used

Manual Review

## Recommendation

Consider limiting the take penalty so it won't worsen the collaterization of the auctioned loan, e.g. limiting `takePenaltyFactor` to `TP / market_price - 1`.
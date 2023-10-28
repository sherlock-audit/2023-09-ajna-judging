Restless White Bee

medium

# Stale interest rate can be used in determining kicker's bond size, reward, and borrower's penalty
## Summary

Kicking operation uses stale interest rate based `npTpRatio` for determining `bondFactor`, and correspondingly taking operation uses stale `bondFactor` based `bpf` for determining kicker's reward. Moreover, the snapshot time for pool rate value, that will be used for `npTpRatio`, can be orchestrated by a borrower, reducing the probability of liquidation at the expense of other borrowers.

## Vulnerability Detail

Pool rate aimed to be the measure of market volatility, but `npTpRatio` isn't updated on kicking, so `bondFactor` that is being set at that point doesn't depend on the current rate, but on the rate as of last loan update, which can happen arbitrarily long ago in the past.

Also, all these operations are directly controlled by the borrower as all the corresponding operations besides after-auction one are borrower initiated: `Loans.update(..., stampNpTpRatio = true)` is called only on borrowing, collateral pulling, or when a borrower directly calls `stampLoan()`, and, lastly, on auction exit, when there is no debt left (`borrower_.t0Debt == 0`).

This way as of moment of kicking the `poolRate_` value the `npTpRatio` was determined with can be arbitrary outdated in general and particularly can be manipulated by borrowed by doing the update via one of these operations when the pool rate is at its lowest with regard to long term volatility of the collateral.

## Impact

Borrower can deliberately avoid updating the `npTpRatio`, i.e. they will not be drawing new debt, pulling collateral or stamping the loan, which are the only operations that update `npTpRatio`. Any user can optimize their operations using many accounts and avoid changing the ones stamped with low enough rates.

The impact is `bondFactor` not fully reflecting the volatility as it's designed to be, and this effect can be orchestrated by the borrower, so it ends up to be more savvy borrowers taking advantage of the less savvy ones, having them as a kind of additional layer of protection, since liquidating the ones who tuned `npTpRatio` is the least profitable, such operations will be performed last, other things being equal.

While the impact is material, being gaming the protocol design in a sense that `npTpRatio` does not represent actual collateral volatility and lower the expectation of kicker reward and the probability of liquidation, it both requires savvy borrowers executing such long term strategy and the existence of non-savvy ones with otherwise similar characteristics, so setting the overall severity to be medium.

## Code Snippet

`npTpRatio` is set via `Loans.update` based on the current `poolRate_`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/internal/Loans.sol#L102-L105

```solidity
        // update Np to Tp ratio of borrower
        if (npTpRatioUpdate_) {
            borrower_.npTpRatio = 1.04 * 1e18 + uint256(PRBMathSD59x18.sqrt(int256(poolRate_))) / 2;
        }
```

`bondFactor` and `bondSize` are set on kicking, that can happen in substantially different market state, so those can be determined by an outdated `npTpRatio`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/helpers/PoolHelper.sol#L418-L429

```solidity
    function _bondParams(
        uint256 borrowerDebt_,
        uint256 npTpRatio_
    ) pure returns (uint256 bondFactor_, uint256 bondSize_) {
        // bondFactor = min((NP-to-TP-ratio - 1)/10, 0.03)
        bondFactor_ = Maths.min(
            0.03 * 1e18,
            (npTpRatio_ - 1e18) / 10
        );

        bondSize_ = Maths.wmul(bondFactor_,  borrowerDebt_);
    }
```

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/KickerActions.sol#L337-L338

```solidity
        // update escrowed bonds balances and get the difference needed to cover bond (after using any kick claimable funds if any)
        kickResult_.amountToCoverBond = _updateEscrowedBonds(auctions_, vars.bondSize);
```

Also, outdated `npTpRatio_` based `amountToCoverBond` can be too small or too big this way.

Since borrower penalty is also `bondFactor_` based it can be outdated too:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/TakerActions.sol#L735-L741

```solidity
        uint256 takePenaltyFactor    = uint256(5 * int256(vars.bondFactor) - vars.bpf + 3) / 4;  // Round up
        uint256 borrowerPrice        = Maths.floorWmul(vars.auctionPrice, Maths.WAD - takePenaltyFactor);

        // To determine the value of quote token removed from a bucket in a bucket take call, we need to account for whether the bond is
        // rewarded or not.  If the bond is rewarded, we need to remove the bond reward amount from the amount removed, else it's simply the 
        // collateral times auction price.
        uint256 netRewardedPrice     = (vars.isRewarded) ? Maths.wmul(Maths.WAD - uint256(vars.bpf), vars.auctionPrice) : vars.auctionPrice;
```

## Tool used

Manual Review

## Recommendation

The reason for current setup is that the driver for updating operation was correct placement a loan in the heap. However, with new `npTpRatio_` formula it now has an additional meaning of tuning the loan state to the current market, so it needs to be called whenever such tuning is needed.

As a most straightforward, possibly non-optimal solution, that can be upgraded later with an introduction of a smaller updating, consider running `Loans.update` with `stampNpTpRatio = true` on kicking, before liquidation parameters being set:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/KickerActions.sol#L321-L324

```solidity
        (vars.bondFactor, vars.bondSize) = _bondParams(
            vars.borrowerDebt,
            borrower.npTpRatio
        );
```
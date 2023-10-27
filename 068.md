Restless White Bee

high

# When pool rates are high some well-capitalized borrowers can be profitably kicked by anyone
## Summary

Using `lenderKick()` even on substantially collaterized borrowers can be possible and profitable when pool rates are high. Borrowers will be penalized to benefit kicker and pool reserves.

Anyone can be a kicker as it is enough to atomically add collateral, do `lenderKick()`, and remove collateral (i.e. bucket price isn't important and there is no need to either be or become a quote funds depositor), so `lenderKick()` be performed as if attacker holds the cumulative deposit of the bucket. The attacker can even gather deposits from many buckets that will not be freezed on kicking this way in one bucket, add enough collateral there, `lenderKick()` target borrower with it, and distribute the deposits back, removing the collateral from those buckets.

This was possible before, but such attack wasn't generally profitable as kicking wasn't as there was no such substantial dependency on pool's rate. But with new formula whenever pool rates are high enough a number of otherwise well-capitalized borrowers can be reached and profitably kicked in this manner.

## Vulnerability Detail

As `neutralPrice` is high when rate is high it is likely to be above market price. Big lenders directly, or anyone, using the fact that bucket collateral isn't frozen by auction, while `lenderKick()` is LP driven only, can use `lenderKick()`, obtaining the collateral at market price, receiving positive kicker's reward, which is the attacker's profit in this case.

Simulateneously borrower's taker penalty will be substantial. For example, since `3.75%` is a maximum loss for borrower given that kicker reward is positive, for pool rate being `45% = 3.75% * 12` (which is not too big, being seen for 1-2 day periods periodically even in the biggest lending pools), which means that borrower will instantly receive a loss worth a month of interest spending.

It is then `npTpRatio = 1.04 + math.sqrt(0.45) / 2 = 1.375..`, so any borrower with `PV(collateral) / PV(debt) < 137.5%` or `LTV > 72.7%` can be profitably kicked. Those ratios represent substantial collaterization which will be deemed safe by the most borrowers, so many will not be aware and will not collaterize more (even if they comfortably could). So receiving a substantial enough penalty will effectively be a form of simultaneous griefing for them.

As anyone can add collateral to the biggest bucket and use `lenderKick()` on it, this vector can be used by any attacker as long as there is a deposit bucket above LUP big enough to reach a target borrower or, as mentioned, if such a bucket can be gathered on the fly from other deposit buckets provided that they will not be affected by `_revertIfAuctionDebtLocked()`.

This can be done atomically and so the price level of those buckets doesn't matter.

Symbolic PoC:

1. flash loan the collateral and obtain longer term funds for kicker's bond,
2. replace quote with collateral in all such quote funds holding deposit buckets,
3. put them all into the one bucket that will not be frozen on auction,
4. add enough collateral there too to have enough LP for the quote funds amount gathered,
5. `lenderKick()` target borrower with this bucket,
6. distribute the affected quote deposits back and remove all the collateral from those and main buckets,
7. repay the flash loan, having only kicker's bond remaining invested.

1-7 are done atomically so exact price levels of these buckets can't be acted on by other market participants.

8. take the auction at the market price,
9. retrieve kicker's bond, repay it, pocket the reward part less funding and transaction costs.

## Impact

The issue here is that such a situation might not be clear for borrowers as per usual metrics they be well capitalized and deem themselves safe, not linking the somewhat yet moderate rise of the interest rate to the ability to be profitably liquidated. Borrowers can receive a substantial take penalty, `takePenaltyFactor in [3%, 3.75%)` (it can go higher when `vars.bpf < 0`, but we rule that out per being unprofitable for kicker).

Since this action is profitable whenever pool rate is high and there is no other substantial prerequisites, the overall probability of it can be deemed high. The borrower's loss is `3%-3.75` of the principal, which can correspond to months of interest, while the borrower intended financing period might be much shorter. Also, a forced sell of a collateral can happen at an undesired market price. This borrower loss impact is material, so placing the overall severity to be high.

The attack can be carried out either for profit only or with an additional griefing purposes.

## Code Snippet

When pool rate is high the `npTpRatio` will also be high:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/internal/Loans.sol#L102-L105

```solidity
        // update Np to Tp ratio of borrower
        if (npTpRatioUpdate_) {
            borrower_.npTpRatio = 1.04 * 1e18 + uint256(PRBMathSD59x18.sqrt(int256(poolRate_))) / 2;
        }
```

`neutralPrice` will be set correspondingly and it can constitute a significant margin above the market price for many well-capitalized borrowers with low enough TPs:

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

`_bpf()` will be positive along with kicker's reward as long as `neutralPrice_ > auctionPrice_ = market_price`:

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
                        int256(neutralPrice_) - thresholdPrice
                    )
                )
            );
        } else {
            ...
        }

        return PRBMathSD59x18.mul(int256(bondFactor_), sign);
    }
```

`lenderKick()` allows collateral depositor to use other depositor's quote funds as `entitledAmount` for `_kick()`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/internal/Buckets.sol#L214-L235

```solidity
    function lpToQuoteTokens(
        ...
    ) internal pure returns (uint256) {
        ...

        // case when there's deposit or collateral and bucket has LP balance
        return Math.mulDiv(
>>          deposit_ * Maths.WAD + bucketCollateral_ * bucketPrice_,
            lp_,
            bucketLP_ * Maths.WAD,
            rounding_
        );
    }
```

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/KickerActions.sol#L135-L185

```solidity
    function lenderKick(
        ...
    ) external returns (
        KickResult memory kickResult_
    ) {
        ...
        vars.bucketDeposit = Deposits.valueAt(deposits_, index_);

        // calculate amount lender is entitled in current bucket (based on lender LP in bucket)
>>      vars.entitledAmount = Buckets.lpToQuoteTokens(
            bucket.collateral,
            bucket.lps,
            vars.bucketDeposit,
            vars.lenderLP,
            vars.bucketPrice,
            Math.Rounding.Down
        );

        // cap the amount entitled at bucket deposit
        if (vars.entitledAmount > vars.bucketDeposit) vars.entitledAmount = vars.bucketDeposit;

        ...

        // kick top borrower
        kickResult_ = _kick(
            ...
            Loans.getMax(loans_).borrower,
            limitIndex_,
>>          vars.entitledAmount
        );
    }
```

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/KickerActions.sol#L273-L296

```solidity
    function _kick(
        ...
>>      uint256 additionalDebt_
    ) internal returns (
        KickResult memory kickResult_
    ) {
        ...
>>      kickResult_.lup = Deposits.getLup(deposits_, poolState_.debt + additionalDebt_);
```

## Tool used

Manual Review

## Recommendation

There is a trader-off between allowing for liquidation of some healthy borrowers and introducing the bigger buffer so that only ones close to the market can be liquidated.

With LP driven nature of `lenderKick()` being the part of design, the root cause of the surface is a somewhat too substantial impact pool rate has on `NP`.

Not all the pools will be liquid and with overall volatility being tied to the pool rate in general. There will be many examples of divergencies over time, even involving substantial enough funds. The ability to act on otherwise healthy borrowers in this situation needs to be controlled for by limiting the dependency of `NP` on the current market state.
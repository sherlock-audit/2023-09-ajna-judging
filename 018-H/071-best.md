Restless White Bee

high

# Reserves can be stolen by settling artificially created bad debt from them
## Summary

It is possible to inflate LUP and pull almost all the collateral from the loan, then settle this artificial bad debt off reserves, stealing all of them by repeating the attack.

## Vulnerability Detail

LUP is determined just in time of the execution, but `removeQuoteToken()` will check for `LUP > HTP = Loans.getMax(loans).thresholdPrice`, so the attack need to circumvent this. 

Attacker acts via 2 accounts under control, one for lending part, one for borrowing, having no previous positions in the given ERC20 pool.
Let's say that pool state is as follows: `100` units of quote token utilized, and another `100` not utilized, for simplicity let's say all `100` are available, no auctions and no bad debt.

1. Get 100 loan at TP = new HTP just below LUP (let's name this loan the manipulated_loan, ML), let's say LUP is such that `150` quote token units worth of collateral was provided (at market price)
2. Add quote funds with `amount = {total utilized deposits} = 200` to a bucket significantly above the market (let's name it ultra_high_bucket, UHB)
3. Remove collateral from the ML up to allowed by elevated LUP = UHB price, say it's 10x market one. I.e. almost all, say remove `140`, leaving `10` quote token units worth of collateral (at market price)
4. Lender kick ML (removing funds from UHB will push LUP low and lender kick will be possible) to revive HTP reading
5. Remove all from UHB except quote funds frozen by the auction, i.e. remove `100`, leave `100`

1-5 go atomically in one tx.

No outside actors will be able to immediately benefit from the resulting situation as:
a. UHB remove quote token is blocked due to the frozen debt
b. take is unprofitable while auction price is above market
c. bucket take is unprofitable while auction price is above UHB price

At this point attacker accounting in net quote tokens worth is:
1. `+100`, `-150`
2. `-200`
3. `+140`
4.  
5. `+100`

Attacker has net `10` units of own capital still invested in the pool.

6. Attacker wait for auction price reaching UHB price, calls `takeBucket` with `depositTake == true`.
The call can go with normal gas price as outside actors will not benefit from the such call and there will be no competition.
`10` quote tokens worth of collateral will be placed to UHB and `98.482` units of quote token removed to cover the debt, `1.518` is the kicker's reward
7. Attacker calls `settlePoolDebt()` which settles the remaining `1.518` of debt from pool reserves as ML has this amount of debt and no collateral
8. Attacker removes all the funds from UHB with both initial and awarded LP shares, receiving `10` quote tokens worth of collateral and `1.518` quote token units of profit

6-8 go atomically in one tx.

At this point attacker accounting in quote tokens is:

6. (+kicker reward of ML stamped NP vs auction clearing UHB price in LP form)
7. 
8. `+11.518`

Attacker has net 0 units of own capital still invested in the pool, and receives `1.518` quote token units of profit.

Profit part is a function of the pool's state, for the number above, assuming `auctionPrice == thresholdPrice = UHB price` and `poolRate_ = 0.05`: `1.518 = bondFactor * 100 = (npTpRatio - 1) / 10 * 100 = (1.04 + math.sqrt(0.05) / 2 - 1) / 10 * 100`, `bpf = bondFactor = takePenaltyFactor`.

## Impact

It can be repeated to drain the whole reserves from the pool over time, i.e. reserves of any pool can be stolen this way.

## Code Snippet

LUP is the only guardian for removing the collateral:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/BorrowerActions.sol#L287-L307

```solidity
        if (vars.pull) {
            // only intended recipient can pull collateral
            if (borrowerAddress_ != msg.sender) revert BorrowerNotSender();

            // calculate LUP only if it wasn't calculated in repay action
            if (!vars.repay) result_.newLup = Deposits.getLup(deposits_, result_.poolDebt);

>>          uint256 encumberedCollateral = Maths.wdiv(vars.borrowerDebt, result_.newLup);
            if (
                borrower.t0Debt != 0 && encumberedCollateral == 0 || // case when small amount of debt at a high LUP results in encumbered collateral calculated as 0
                borrower.collateral < encumberedCollateral ||
                borrower.collateral - encumberedCollateral < collateralAmountToPull_
            ) revert InsufficientCollateral();

            // stamp borrower Np to Tp ratio when pull collateral action
            vars.stampNpTpRatio = true;

            borrower.collateral -= collateralAmountToPull_;

            result_.poolCollateral -= collateralAmountToPull_;
        }
```

The root cause is that bad debt is artificially created, with lender, borrower and kicker being controlled by the attacker, bad debt is then settled from the reserves:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L143-L146

```solidity
            // settle debt from reserves (assets - liabilities) if reserves positive, round reserves down however
            if (assets > liabilities) {
                borrower.t0Debt -= Maths.min(borrower.t0Debt, Maths.floorWdiv(assets - liabilities, poolState_.inflator));
            }
```

Kicking will revive the HTP as `_kick()` removing target borrower from loans:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/KickerActions.sol#L340-L341

```solidity
        // remove kicked loan from heap
        Loans.remove(loans_, borrowerAddress_, loans_.indices[borrowerAddress_]);
```

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/base/Pool.sol#L225-L249

```solidity
    function removeQuoteToken(
        ...
    ) external override nonReentrant returns (uint256 removedAmount_, uint256 redeemedLP_) {
        ...

        uint256 newLup;
        (
            removedAmount_,
            redeemedLP_,
            newLup
        ) = LenderActions.removeQuoteToken(
            ...
            RemoveQuoteParams({
                maxAmount:      Maths.min(maxAmount_, _availableQuoteToken()),
                index:          index_,
>>              thresholdPrice: Loans.getMax(loans).thresholdPrice
            })
        );
```

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/LenderActions.sol#L434-L436

```solidity
        lup_ = Deposits.getLup(deposits_, poolState_.debt);

>>      uint256 htp = Maths.wmul(params_.thresholdPrice, poolState_.inflator);
```

## Tool used

Manual Review

## Recommendation

Consider introducing a buffer representing the expected kicker reward in addition to LUP, so this part of the loan will remain in the pool.
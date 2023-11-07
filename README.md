# Issue H-1: Subsequent takes increase next kicker reward, allowing total kicker reward to be artificially amplified by splitting take into a batch 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/69 

## Found by 
hyh

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

# Issue H-2: Reserves can be stolen by settling artificially created bad debt from them 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/71 

## Found by 
hyh

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

# Issue M-1: Wrong auctionPrice used in calculating BPF to determine bond reward (or penalty) 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/16 

## Found by 
CL001
According to the Ajna Protocol Whitepaper(section  7.4.2 Deposit Take),in example:

"Instead of Eve purchasing collateral using take, she will call deposit take. Note auction goes through 6 twenty minute halvings, followed by 2 two hour halvings, and then finally 22.8902 minutes (0.1900179029 * 120min) of a two hour halving. After 6.3815 hours (6*20 minutes +2*120 minutes + 22.8902 minutes), the auction price has fallen to 312, 998. 784 · 2 ^ −(6+2+0.1900179) =1071.77
which is below the price of 1100 where Carol has 20000 deposit.
Deposit take will purchase the collateral using the deposit at price 1100 and the neutral price is 1222.6515, so the BPF calculation is:
BPF = 0.011644 * 1222.6515-1100 / 1222.6515-1050 = 0.008271889129“.

As described in the whiterpaper, in the case of user who calls Deposit Take, the correct approach to calculating BPF is to use bucket price(1100  instead of 1071.77) when auctionPrice < bucketPrice .

## Vulnerability Detail
1.bucketTake() function in TakerActions.sol calls the  _takeBucket()._prepareTake() to calculate the BPF.
https://github.com/sherlock-audit/2023-09-ajna/blob/87abfb6a9150e5df3819de58cbd972a66b3b50e3/ajna-core/src/libraries/external/TakerActions.sol#L416
https://github.com/sherlock-audit/2023-09-ajna/blob/87abfb6a9150e5df3819de58cbd972a66b3b50e3/ajna-core/src/libraries/external/TakerActions.sol#L688

2.In _prepareTake() function,the BPF is calculated using vars.auctionPrice which is calculated by _auctionPrice() function.
```solidity
function _prepareTake(
        Liquidation memory liquidation_,
        uint256 t0Debt_,
        uint256 collateral_,
        uint256 inflator_
    ) internal view returns (TakeLocalVars memory vars) {
 ........
        vars.auctionPrice = _auctionPrice(liquidation_.referencePrice, kickTime);
        vars.bondFactor   = liquidation_.bondFactor;
        vars.bpf          = _bpf(
            vars.borrowerDebt,
            collateral_,
            neutralPrice,
            liquidation_.bondFactor,
            vars.auctionPrice
        );
```
3.The _takeBucket() function made a judgment after _prepareTake() 
```solidity
 // cannot arb with a price lower than the auction price
if (vars_.auctionPrice > vars_.bucketPrice) revert AuctionPriceGtBucketPrice();
// if deposit take then price to use when calculating take is bucket price
if (params_.depositTake) vars_.auctionPrice = vars_.bucketPrice;
```

so the root cause of this issue is that  in a scenario where a user calls Deposit Take(params_.depositTake ==true) ,BPF will calculated base on vars_.auctionPrice  instead of bucketPrice. 

And then,BPF is used to calculate takePenaltyFactor, borrowerPrice , netRewardedPrice and bondChange in the  _calculateTakeFlowsAndBondChange() function，and direct impact on  the _rewardBucketTake() function.
https://github.com/sherlock-audit/2023-09-ajna/blob/87abfb6a9150e5df3819de58cbd972a66b3b50e3/ajna-core/src/libraries/external/TakerActions.sol#L724

https://github.com/sherlock-audit/2023-09-ajna/blob/87abfb6a9150e5df3819de58cbd972a66b3b50e3/ajna-core/src/libraries/external/TakerActions.sol#L640
```solidity
 vars_ = _calculateTakeFlowsAndBondChange(
            borrower_.collateral,
            params_.inflator,
            params_.collateralScale,
            vars_
        );
.............
_rewardBucketTake(
            auctions_,
            deposits_,
            buckets_,
            liquidation,
            params_.index,
            params_.depositTake,
            vars_
        );
```


## Impact
Wrong auctionPrice used in calculating BFP which subsequently influences the _calculateTakeFlowsAndBondChange() and _rewardBucketTake() function will result in bias .


Following the example of the Whitepaper(section 7.4.2 Deposit Take)：
BPF = 0.011644 * (1222.6515-1100 / 1222.6515-1050) = 0.008271889129
The collateral purchased is min{20, 20000/(1-0.00827) * 1100, 21000/(1-0.01248702772 )* 1100)} which is 18.3334 unit of ETH .Therefore, 18.3334 ETH are moved from the loan into the claimable collateral of bucket 1100, and the deposit is reduced to 0. Dave is awarded LPB in that bucket worth 18. 3334 · 1100 · 0. 008271889129 = 166. 8170374 𝐷𝐴𝐼.
The debt repaid is 19914.99407 DAI

----------------------------------------------

Based on the current implementations:
BPF = 0.011644 * (1222.6515-1071.77 / 1222.6515-1050) = 0.010175.
TPF = 5/4*0.011644 - 1/4 *0.010175 = 0.01201125.
The collateral purchased is 18.368 unit of ETH.
The debt repaid is 20000 * (1-0.01201125) / (1-0.010175) = 19962.8974DAI
Dave is awarded LPB in that bucket worth 18. 368 · 1100 · 0. 010175 = 205.58 𝐷𝐴𝐼.

So,Dave earn more rewards（38.703 DAI） than he should have.

As the Whitepaper says:
"
the caller would typically have other motivations. She might have called it because she is Carol, who wanted to buy the ETH but not add additional DAI to the contract. She might be Bob, who is looking to get his debt repaid at the best price. She might be Alice, who is looking to avoid bad debt being pushed into the contract. She also might be Dave, who is looking to ensure a return on his liquidation bond."





## Code Snippet
https://github.com/sherlock-audit/2023-09-ajna/blob/87abfb6a9150e5df3819de58cbd972a66b3b50e3/ajna-core/src/libraries/external/TakerActions.sol#L416

## Tool used
Manual Review

## Recommendation
In the case of Deposit Take calculate BPF using  bucketPrice instead of auctionPrice .



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**n33k** commented:
>  "`if (params_.depositTake) vars_.auctionPrice = vars_.bucketPrice;` made "



# Issue M-2: HPB may be incorrectly bankrupt due to use of unscaled value in `_forgiveBadDebt` 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/25 

## Found by 
0xkaden

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

# Issue M-3: First pool borrower pays extra interest 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/26 

## Found by 
0xkaden

There exists an exception in the interest logic in which the action of borrowing from a pool for the first time (or otherwise when there is 0 debt) does not trigger the inflator to update. As a result, the borrower's interest effectively started accruing at the last time the inflator was updated, before they even borrowed, causing them to pay more interest than intended.

## Vulnerability Detail

For any function in which the current interest rate is important in a pool, we compute interest updates by accruing with `_accruePoolInterest` at the start of the function, then execute the main logic, then update the interest state accordingly with `_updateInterestState`. See below a simplified example for `ERC20Pool.drawDebt`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/ERC20Pool.sol#L130
```solidity
function drawDebt(
    address borrowerAddress_,
    uint256 amountToBorrow_,
    uint256 limitIndex_,
    uint256 collateralToPledge_
) external nonReentrant {
    PoolState memory poolState = _accruePoolInterest();

   ...

    DrawDebtResult memory result = BorrowerActions.drawDebt(
        auctions,
        deposits,
        loans,
        poolState,
        _availableQuoteToken(),
        borrowerAddress_,
        amountToBorrow_,
        limitIndex_,
        collateralToPledge_
    );

    ...

    // update pool interest rate state
    _updateInterestState(poolState, result.newLup);

   ...
}
```

When accruing interest in `_accruePoolInterest`, we only update the state if `poolState_.t0Debt != 0`. Most notably, we don't set `poolState_.isNewInterestAccrued`. See below simplified logic from `_accruePoolInterest`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/base/Pool.sol#L552
```solidity
// check if t0Debt is not equal to 0, indicating that there is debt to be tracked for the pool
if (poolState_.t0Debt != 0) {
    ...

    // calculate elapsed time since inflator was last updated
    uint256 elapsed = block.timestamp - inflatorState.inflatorUpdate;

    // set isNewInterestAccrued field to true if elapsed time is not 0, indicating that new interest may have accrued
    poolState_.isNewInterestAccrued = elapsed != 0;

    ...
}
```

Of course before we actually update the state from the first borrow, the debt of the pool is 0, and recall that `_accruePoolInterest` runs before the main state changing logic of the function in `BorrowerActions.drawDebt`.

After executing the main state changing logic in `BorrowerActions.drawDebt`, where we update state, including incrementing the pool and borrower debt as expected, we run the logic in `_updateInterestState`. Here we update the inflator if either `poolState_.isNewInterestAccrued` or `poolState_.debt == 0`.

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/base/Pool.sol#L686
```solidity
// update pool inflator
if (poolState_.isNewInterestAccrued) {
    inflatorState.inflator       = uint208(poolState_.inflator);
    inflatorState.inflatorUpdate = uint48(block.timestamp);
// if the debt in the current pool state is 0, also update the inflator and inflatorUpdate fields in inflatorState
// slither-disable-next-line incorrect-equality
} else if (poolState_.debt == 0) {
    inflatorState.inflator       = uint208(Maths.WAD);
    inflatorState.inflatorUpdate = uint48(block.timestamp);
}
```

The problem here is that since there was no debt at the start of the function, `poolState_.isNewInterestAccrued` is false and since there is debt now at the end of the function, `poolState_.debt == 0` is also false. As a result, the inflator is not updated. Updating the inflator here is paramount since it effectively marks a starting time at which interest accrues on the borrowers debt. Since we don't update the inflator, the borrowers debt effectively started accruing interest at the time of the last inflator update, which is an arbitrary duration.

We can prove this vulnerability by modifying `ERC20PoolBorrow.t.sol:testPoolBorrowAndRepay` to skip 100 days before initially drawing debt:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/tests/forge/unit/ERC20Pool/ERC20PoolBorrow.t.sol#L94
```solidity
function testPoolBorrowAndRepay() external tearDown {
    // check balances before borrow
    assertEq(_quote.balanceOf(address(_pool)), 50_000 * 1e18);
    assertEq(_quote.balanceOf(_lender),        150_000 * 1e18);

    // @audit skip 100 days to break test
    skip(100 days);

    _drawDebt({
        from: _borrower,
        borrower: _borrower,
        amountToBorrow: 21_000 * 1e18,
        limitIndex: 3_000,
        collateralToPledge: 100 * 1e18,
        newLup: 2_981.007422784467321543 * 1e18
    });

    ...
}
```

Unlike the result without skipping time before drawing debt, the test fails with output logs being off by amounts roughly corresponding to the unexpected interest.
![image](https://github.com/sherlock-audit/2023-09-ajna-kadenzipfel/assets/30579067/6196d147-ff67-4781-aa76-cae408be759d)

## Impact

First borrower **always** pays extra interest, with losses depending upon time between adding liquidity and drawing debt and amount of debt drawn.

Note also that there's an attack vector here in which the liquidity provider can intentionally create and fund the pool a long time before announcing it, causing the initial borrower to lose a significant amount to interest.

## Code Snippet

See 'Vulnerability Detail' section for snippets.

## Tool used

- Manual Review
- Forge

## Recommendation

When checking whether the debt of the pool is 0 to determine whether to reset the inflator, it should not only check whether the debt is 0 at the end of execution, but also whether the debt was 0 before execution. To do so, we should cache the debt at the start of the function and modify the `_updateInterestState` logic to be something like:

```solidity
// update pool inflator
if (poolState_.isNewInterestAccrued) {
    inflatorState.inflator       = uint208(poolState_.inflator);
    inflatorState.inflatorUpdate = uint48(block.timestamp);
// if the debt in the current pool state is 0, also update the inflator and inflatorUpdate fields in inflatorState
// slither-disable-next-line incorrect-equality
// @audit reset inflator if no debt before execution
} else if (poolState_.debt == 0 || debtBeforeExecution == 0) {
    inflatorState.inflator       = uint208(Maths.WAD);
    inflatorState.inflatorUpdate = uint48(block.timestamp);
}
```

# Issue M-4: Function `_indexOf` will cause a settlement to revert if `auctionPrice > MAX_PRICE` 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/30 

## Found by 
santipu\_

In ERC721 pools, when an auction is settled while `auctionPrice > MAX_PRICE` and borrower has some fraction of collateral (e.g. `0.5e18`) the settlement will always revert until enough time has passed so `auctionPrice` lowers below `MAX_PRICE`, thus causing a temporary DoS. 

## Vulnerability Detail

In ERC721 pools, when a settlement occurs and the borrower still have some fraction of collateral, that fraction is allocated in the bucket with a price closest to `auctionPrice` and the borrower is proportionally compensated with LPB in that bucket.

In order to calculate the index of the bucket closest in price to `auctionPrice`, the `_indexOf` function is called. The first line of that function is outlined below:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/helpers/PoolHelper.sol#L78
```solidity
if (price_ < MIN_PRICE || price_ > MAX_PRICE) revert BucketPriceOutOfBounds();
```

The `_indexOf` function will revert if `price_` (provided as an argument) is below `MIN_PRICE` or above `MAX_PRICE`. This function is called from `_settleAuction`, here is a snippet of that:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L234

```solidity
function _settleAuction(
    AuctionsState storage auctions_,
    mapping(uint256 => Bucket) storage buckets_,
    DepositsState storage deposits_,
    address borrowerAddress_,
    uint256 borrowerCollateral_,
    uint256 poolType_
) internal returns (uint256 remainingCollateral_, uint256 compensatedCollateral_) {

    // ...

            uint256 auctionPrice = _auctionPrice(
                auctions_.liquidations[borrowerAddress_].referencePrice,
                auctions_.liquidations[borrowerAddress_].kickTime
            );

            // determine the bucket index to compensate fractional collateral
>>>         bucketIndex = auctionPrice > MIN_PRICE ? _indexOf(auctionPrice) : MAX_FENWICK_INDEX;

    // ...
}
```

The `_settleAuction` function first calculates the `auctionPrice` and then it gets the index of the bucket with a price closest to `bucketPrice`. If `auctionPrice` results to be bigger than `MAX_PRICE`, then the `_indexOf` function will revert and the entire settlement will fail. 

In certain types of pools where one asset has an extremely low market price and the other is valued really high, the resulting prices at an auction can be so high that is not rare to see an `auctionPrice > MAX_PRICE`.

The `auctionPrice` variable is computed from `referencePrice` and it goes lower through time until 72 hours have passed. Also, `referencePrice` can be much higher than `MAX_PRICE`, as outline in `_kick`:

```solidity
vars.referencePrice = Maths.min(Maths.max(vars.htp, vars.neutralPrice), MAX_INFLATED_PRICE);
```

The value of `MAX_INFLATED_PRICE` is exactly `50 * MAX_PRICE` so a `referencePrice` bigger than `MAX_PRICE` is totally possible. 

In auctions where `referencePrice` is bigger than `MAX_PRICE` and the auction is settled in a low time frame, `auctionPrice` will be also bigger than `MAX_PRICE` and that will cause the entire transaction to revert. 

## Impact

When the above conditions are met, the auction won't be able to settle until `auctionPrice` lowers below `MAX_PRICE`.

In ERC721 pools with a high difference in assets valuation, there is no low-probability prerequisites and the impact will be a violation of the system design, as well as the potential losses for the kicker of that auction, so setting severity to be high

## Code Snippet

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L234

## Tool used

Manual Review

## Recommendation

It's recommended to change the affected line of `_settleAuction` in the following way:

```diff
-   bucketIndex = auctionPrice > MIN_PRICE ? _indexOf(auctionPrice) : MAX_FENWICK_INDEX;
+   if(auctionPrice < MIN_PRICE){
+       bucketIndex = MAX_FENWICK_INDEX;
+   } else if (auctionPrice > MAX_PRICE) {
+       bucketIndex = 1;
+   } else {
+       bucketIndex = _indexOf(auctionPrice);
+   }
```



## Discussion

**neeksec**

Valid finding.
Set to `Medium` because I don't see `the potential losses for the kicker of that auction`. This revert happens under extreme market condition and the it could recovery as time elapses.

# Issue M-5: Unsafe truncation casting is used for a number of state variables, including uncapped ones 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/46 

## Found by 
hyh

`inflator`, `bondSize`, `t0ThresholdPrice` are truncated in the logic without overflow checks.

## Vulnerability Detail

Although the probabilities of reaching the corresponding limits are very small, the consequences of material truncations is pool accounting corruption.

## Impact

If truncation does happen, then it will facilitate massive losses for pool users, say `inflator` one will reset the balances. Future code evolution inheriting the logic might end up having substantially bigger probabilities of reaching the type limits.

## Code Snippet

`poolState_.inflator` isn't capped:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/base/Pool.sol#L542-L573

```solidity
    function _accruePoolInterest() internal returns (PoolState memory poolState_) {
        ...

            // if new interest may have accrued, call accrueInterest function and update inflator and debt fields of poolState_ struct
            if (poolState_.isNewInterestAccrued) {
>>              (uint256 newInflator, uint256 newInterest) = PoolCommons.accrueInterest(
                    ...
                );
>>              poolState_.inflator = newInflator;
                // After debt owed to lenders has accrued, calculate current debt owed by borrowers
                poolState_.debt = Maths.wmul(poolState_.t0Debt, poolState_.inflator);
```

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/PoolCommons.sol#L220-L231

```solidity
    function accrueInterest(
        EmaState      storage emaParams_,
        DepositsState storage deposits_,
        PoolState calldata poolState_,
        uint256 thresholdPrice_,
        uint256 elapsed_
    ) external returns (uint256 newInflator_, uint256 newInterest_) {
        // Scale the borrower inflator to update amount of interest owed by borrowers
>>      uint256 pendingFactor = PRBMathUD60x18.exp((poolState_.rate * elapsed_) / 365 days);

        // calculate the highest threshold price
>>      newInflator_ = Maths.wmul(poolState_.inflator, pendingFactor);
```

At some point very distant point it will be silently truncated with disastrous debt state resetting outcome:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/base/Pool.sol#L678-L687

```solidity
    function _updateInterestState(
        PoolState memory poolState_,
        uint256 lup_
    ) internal {

        PoolCommons.updateInterestState(interestState, emaState, deposits, poolState_, lup_);

        // update pool inflator
        if (poolState_.isNewInterestAccrued) {
            inflatorState.inflator       = uint208(poolState_.inflator);
```

Also, for completness, notice that `vars.bondChange` is truncated in `_rewardTake()`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/TakerActions.sol#L565-L583

```solidity
    function _rewardTake(
        AuctionsState storage auctions_,
        Liquidation storage liquidation_,
        TakeLocalVars memory vars
    ) internal {
        if (vars.isRewarded) {
            // take is below neutralPrice, Kicker is rewarded
>>          liquidation_.bondSize                 += uint160(vars.bondChange);
            auctions_.kickers[vars.kicker].locked += vars.bondChange;
            auctions_.totalBondEscrowed           += vars.bondChange;
        } else {
            // take is above neutralPrice, Kicker is penalized
            vars.bondChange = Maths.min(liquidation_.bondSize, vars.bondChange);

>>          liquidation_.bondSize                 -= uint160(vars.bondChange);
            auctions_.kickers[vars.kicker].locked -= vars.bondChange;
            auctions_.totalBondEscrowed           -= vars.bondChange;
        }
    }
```

And in `_rewardBucketTake()`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/TakerActions.sol#L652-L660

```solidity
        } else {
            // take is above neutralPrice, Kicker is penalized
            vars.bondChange = Maths.min(liquidation_.bondSize, vars.bondChange);

>>          liquidation_.bondSize -= uint160(vars.bondChange);

            auctions_.kickers[vars.kicker].locked -= vars.bondChange;
            auctions_.totalBondEscrowed           -= vars.bondChange;
        }
```

`t0ThresholdPrice` is truncated in `update()`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/internal/Loans.sol#L83-L94

```solidity
        uint256 t0ThresholdPrice = activeBorrower ? Maths.wdiv(borrower_.t0Debt, borrower_.collateral) : 0;

        // loan not in auction, update threshold price and position in heap
        if (!inAuction_ ) {
            // get the loan id inside the heap
            uint256 loanId = loans_.indices[borrowerAddress_];
            if (activeBorrower) {
                // revert if threshold price is zero
                if (t0ThresholdPrice == 0) revert ZeroThresholdPrice();

                // update heap, insert if a new loan, update loan if already in heap
>>              _upsert(loans_, borrowerAddress_, loanId, uint96(t0ThresholdPrice));
```

## Tool used

Manual Review

## Recommendation

For the sake of unification and eliminating these considerations consider using `SafeCast` in all the occurrences above, e.g.:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/KickerActions.sol#L396-L411

```solidity
    function _recordAuction(
        ...
    ) internal {
        // record liquidation info
        liquidation_.kicker         = msg.sender;
        liquidation_.kickTime       = uint96(block.timestamp);
        liquidation_.referencePrice = SafeCast.toUint96(referencePrice_);
        liquidation_.bondSize       = SafeCast.toUint160(bondSize_);
        liquidation_.bondFactor     = SafeCast.toUint96(bondFactor_);
        liquidation_.neutralPrice   = SafeCast.toUint96(neutralPrice_);
```

# Issue M-6: lenderKick incorrectly sets LUP 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/49 

## Found by 
0xkaden

`KickerActions.lenderKick` retrieves what the LUP would be if the lender's deposit were removed to validate collateralization of the borrower being kicked. The method doesn't actually add to the deposit but returns the incorrect LUP where it is later incorrectly used to update the interest rate.

## Vulnerability Detail

In `KickerActions.lenderKick`, we compute the `entitledAmount` of quote tokens if the lender were to withdraw their whole position. We pass this value as `additionalDebt_` to `_kick` where it allows us to compute what the LUP would be if the lender removed their position. The function then proceeds to validate that the new LUP would leave the borrower undercollateralized, and if so, kick that borrower. 

The problem is that we then return the computed LUP even though we aren't actually removing the lender's quote tokens. In `Pool.lenderKick`, we then pass this incorrect LUP to `_updateInterestState` where it is used to incorrectly update the `lupt0DebtEma`, which is used to calculate the interest rate, leading to an incorrect rate.

## Impact

Broken core invariant related to interest rate calculation. Impact on interest rate is dependent upon size of lender's position relative to total deposit size.

## Code Snippet

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/KickerActions.sol#L296
```solidity
// add amount to remove to pool debt in order to calculate proposed LUP
// for regular kick this is the currrent LUP in pool
// for provisional kick this simulates LUP movement with additional debt
kickResult_.lup = Deposits.getLup(deposits_, poolState_.debt + additionalDebt_);
```

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/base/Pool.sol#L363
```solidity
_updateInterestState(poolState, result.lup);
```

## Tool used

Manual Review

## Recommendation

Calculate the actual LUP as `kickResult_.lup`, then calculate the simulated LUP separately with an unrelated variable.



## Discussion

**ith-harvey**

Does not meet validity criteria for `Medium`. Will fix it.

**neeksec**

Keep `Medium` since this issue leads to an incorrect interest rate.

# Issue M-7: Unutilized deposit fee can be avoided or accidentally double paid 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/51 

## Found by 
0xkaden

A check in `moveQuoteToken` to determine whether the unutilized deposit fee must be paid allows users to avoid paying the fee at all and can unintentionally cause users to pay the fee more than once.

## Vulnerability Detail

When moving quote tokens from one bucket to another with `moveQuoteToken`, a deposit fee is charged if `vars.fromBucketPrice >= lup_ && vars.toBucketPrice < lup_`. This is to ensure that if a user already paid the fee that they don't double pay and if they hadn't yet paid the fee but they should for moving into an unutilized bucket that they do here.

The problem with this logic is the LUP is a variable price and as such it can result in the following unexpected effects:

1) If a lender deposits initially to a utilized bucket but then the LUP moves up above it, they can then move to any other unutilized bucket, never having had paid a fee.

2) If a lender deposits initially to an unutilized bucket but then the LUP moves down below it, if they move their tokens to an unutilized bucket, they will be charged the unutilized deposit fee twice.

Both of these cases are contrary to the intended effect of this check. Furthermore, lenders can manipulate 1) to cheaply avoid paying a utilization fee while depositing in any unutilized bucket as follows:
- Take out just enough debt to push LUP into the next bucket
- addQuoteTokens at new LUP bucket
- Repay debt
  - We are now in an unutilized bucket without paying a fee
- moveQuoteTokens to any unutilized bucket
  - We still don't have to pay the fee

## Impact

Loss of user funds and allowing users to avoid paying fees. 

Section 4.2 of the [whitepaper](https://www.ajna.finance/pdf/Ajna_Protocol_Whitepaper_10-12-2023.pdf?ref=ajna-protocol-news.ghost.io) notes that this fee is used to mitigate MEV attacks, which as a result of this finding are likely not prevented.

## Code Snippet

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/LenderActions.sol#L292
```solidity
// apply unutilized deposit fee if quote token is moved from above the LUP to below the LUP
// @audit what if from bucket used to be above LUP so they never pay the fee?
//        or what if they paid a fee initially than LUP moved down and now they have to pay again?
if (vars.fromBucketPrice >= lup_ && vars.toBucketPrice < lup_) {
    if (params_.revertIfBelowLup) revert PriceBelowLUP();

    movedAmount_ = Maths.wmul(movedAmount_, Maths.WAD - _depositFeeRate(poolState_.rate));
}
```

## Tool used

Manual Review

## Recommendation

Include a boolean in storage as to whether the lender was already charged the fee for that particular bucket, or simply charge the unutilized deposit fee every time the `toBucketPrice < lup_`, regardless of the previous bucket price.



## Discussion

**sherlock-admin2**

1 comment(s) were left on this issue during the judging contest.

**n33k** commented:
>  "both are consistent with the whitepaper"



**neeksec**

Valid finding.

# Issue M-8: settlePoolDebt places the collateral to the lowest bucket, allowing for stealing it with back-running 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/52 

## Found by 
hyh

SettlerActions's `_settleAuction()` and `_settlePoolDebtWithDeposit()` run from `settlePoolDebt()` always put remainder and compensated collateral of a borrower in the lowest bucket, where is can be straightforwardly stolen by back-running the function.

## Vulnerability Detail

For the borrower these funds are accessible no matter what bucket they are placed to. In the same time, when the bucket is below the market, these funds are free for anyone to obtain, pocketing the difference. SettlerActions's logic places them into the lowest bucket, despite being able to use quote deposit distribution information to gauge the price. Also, it might be a successful auction at the start and what is settled is only the remainder, so the executed auction price can be used. The issue is that setting the price too small allows for straightforward stealing, while using price too big only creates harmless above market `ask` from the borrower.

The fact that there are no deposits left (for `_settlePoolDebtWithDeposit()`) or that some part of the debt wasn't bought on auction (for `_settleAuction()` run from `settlePoolDebt()`) doesn't form enough grounds for estimating the fair price being equal to the lowest bucket's one. Lack of deposits can be of a technical nature (for example, drawing debt creates immediate debt overhang formed by a borrowing fee), while leaving minimal debt not bought at any price can be due to, as an example: pool doesn't have many participants overall, say some not too liquid NFT with small volumes, there was an initial taker, who left close to minimal debt due to oversight/own liquidity reasons (or knowing that there can be a situation of lack of interest and trying to exploit it), minimal debt is small in this pool, gas prices rocketed for some reasons, so taking the remaining part was not profitable, so the auction had no bids for the remainder and has to be cleared later with `settlePoolDebt()` in a calmer market.

## Impact

Borrower will lose all the placed collateral as anyone can run its extraction right after the settlement.

## Code Snippet

Bucket for collateral remainder is determined as if auction is taking place now:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L206-L244

```solidity
    function _settleAuction(
        ...
    ) internal returns (uint256 remainingCollateral_, uint256 compensatedCollateral_) {

        if (poolType_ == uint8(PoolType.ERC721)) {
            uint256 lp;
            uint256 bucketIndex;

            // floor collateral of borrower
            remainingCollateral_ = (borrowerCollateral_ / Maths.WAD) * Maths.WAD;

            // if there's fraction of NFTs remaining then reward difference to borrower as LP in auction price bucket
            if (remainingCollateral_ != borrowerCollateral_) {

                // calculate the amount of collateral that should be compensated with LP
                compensatedCollateral_ = borrowerCollateral_ - remainingCollateral_;

>>              uint256 auctionPrice = _auctionPrice(
                    auctions_.liquidations[borrowerAddress_].referencePrice,
                    auctions_.liquidations[borrowerAddress_].kickTime
                );

                // determine the bucket index to compensate fractional collateral
>>              bucketIndex = auctionPrice > MIN_PRICE ? _indexOf(auctionPrice) : MAX_FENWICK_INDEX;

                // deposit collateral in bucket and reward LP to compensate fractional collateral
                lp = Buckets.addCollateral(
                    buckets_[bucketIndex],
                    borrowerAddress_,
                    Deposits.valueAt(deposits_, bucketIndex),
                    compensatedCollateral_,
                    _priceAt(bucketIndex)
                );
            }
```

When `SettlerActions._settleAuction()` is called from `TakerActions._takeLoan()` it is reasonable, but when it's called from `SettlerActions.settlePoolDebt()` and collateral isn't zero it is always `block.timestamp - kickTime > 72 hours`, i.e. auction price above will be close to zero as auction was ended:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L100-L113

```solidity
    function settlePoolDebt(
        AuctionsState storage auctions_,
        mapping(uint256 => Bucket) storage buckets_,
        DepositsState storage deposits_,
        LoansState storage loans_,
        ReserveAuctionState storage reserveAuction_,
        PoolState calldata poolState_,
        SettleParams memory params_
    ) external returns (SettleResult memory result_) {
        uint256 kickTime = auctions_.liquidations[params_.borrower].kickTime;
        if (kickTime == 0) revert NoAuction();

        Borrower memory borrower = loans_.borrowers[params_.borrower];
>>      if ((block.timestamp - kickTime <= 72 hours) && (borrower.collateral != 0)) revert AuctionNotClearable();
```

Similarly, it is the lowest bucket whenever all deposits happen to be used:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L347-L352

```solidity
            (vars.index, , vars.scale) = Deposits.findIndexAndSumOfSum(deposits_, 1);
            vars.hpbUnscaledDeposit    = Deposits.unscaledValueAt(deposits_, vars.index);
            vars.unscaledDeposit       = vars.hpbUnscaledDeposit;
            vars.price                 = _priceAt(vars.index);

>>          if (vars.unscaledDeposit != 0) {
```

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L410-L421

```solidity
>>          } else {
                // Deposits in the tree is zero, insert entire collateral into lowest bucket 7388
                Buckets.addCollateral(
>>                  buckets_[vars.index],
                    params_.borrower,
                    0,  // zero deposit in bucket
                    remainingCollateral_,
>>                  vars.price
                );
                // entire collateral added into bucket, no borrower pledged collateral remaining
                remainingCollateral_ = 0;
            }
```

In both cases this lowest bucket might not correctly correspond to the market price of collateral and the funds placed this way can be stolen from the borrower by back-running such transaction with adding quote and remove collateral funds from this near zero bucket.

## Tool used

Manual Review

## Recommendation

As in both cases there were a settlement process across the deposit buckets taking place before, its information, say the average bid price, can be used for the placement, along with the latest take price, when available.

E.g. consider determining the best estimation of bid price this way, when available, and putting the funds in SettlerActions.sol#L206-L244 and SettlerActions.sol#L410-L421 there. Borrower will be able to remove them from any bucket, while the ability to steal from them will be reduced this way. I.e. if such bucket be above market it will just act as above-market offer, which is harmless.

# Issue M-9: BucketTake rewards can go to an insolvent bucket 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/59 

## Found by 
hyh

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

# Issue M-10: Settlement can reach the state when writing off the debt exhausts all deposits and current reserves, freezing the pool with a not cleared auction 

Source: https://github.com/sherlock-audit/2023-09-ajna-judging/issues/63 

## Found by 
hyh

A situation of total debt exceeding combined total deposits and reserves can be reached, in which case debt settlement will not be able to clear bad debt fully, and pool will be stuck in the `auction not settled` state until an arbitrary large donation of a missing part be made directly to pool's balance, replenishing the reserves.

## Vulnerability Detail

There might be two ways for making total debt exceeding the sum of total deposits and reserves:

A. Pool having near zero interest rate and `_borrowerFeeRate` making debt exceeding total deposits due to BorrowerActions#161 logic, where debt is added in advance (instead of borrower receiving the partial sum, the immediate fee debt is added, while no regular interest accrued yet). As floor division is used for assets and reserve write down in SettlerActions#145, it might be just some dust being left not addressed.

B. Any depositor can deliberately or by accident move some of the deposit to reserves by moving quote tokens under LUP: deposit fee is transferred to reserves this way.

In both cases it is just a transfer from deposits to reserves, but those can be auctioned and removed from the pool, creating a deficit.

Schematic PoC:

1. Two depositors, 50 units of quote token each, one borrower, some illiquid NFT as a collateral.
100 units of initial deposit, all lent out, some interest was accrued and let's say it's 110 debt now.
One depositor performed several deposit moves to under the LUP and is penalized cumulatively by 5, so their deposit remained at 50, while the other is 55.
Reserve auction happens and removes 4.5 from reserves.
So it is 105 in deposits, 110 in debt, debt HTP below LUP, 0.5 in reserves

2. Market drops and is messy for a while, particularly no buyers for that NFT for few days at any price

3. As market drops one lender wants out and performs lenderKick() on the borrower

4. No bidders for the whole length of auction along with (2) (for simplicity, the end result wouldn't change if this be replaced with 'some bids around LUP' as that's essentially what settlement does. The key here is that no material outside capital was brought in, which is reasonable assumption given (2))

5. One of the lenders calls settle() for the borrower

6. _settlePoolDebtWithDeposit() step 1 settles 105 debt with the same amount of deposits at their bucket price, borrower will still have collateral as HTP was below LUP

7. Since there are no deposits left it puts all the collateral into lowest 7388 bucket

8. The remaining 0.5 of the debt is written off reserves, which the remaining 4.5 not being able to be cleared. This sum needs to be donated directly to the pool in order for it to be unstuck

## Impact

All operations requiring `_revertIfAuctionClearable()` will be blocked. One way to unblock them looks to be a donation of the missing part directly to pool's balance. This donation can be material and the freeze can take place for a while.

Due to both fees mechanics working in the same direction with regard to pool's deposit to debt balance, estimating overall probability of reaching the described state to be medium, while pool blocking impact looks to be high, so setting overall severity to be high.

## Code Snippet

BorrowerActions' `drawDebt()` creates new debt in advance:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/BorrowerActions.sol#L160-L161

```solidity
            // t0 debt change is t0 amount to borrow plus the origination fee
            vars.t0DebtChange = Maths.wmul(vars.t0BorrowAmount, _borrowFeeRate(poolState_.rate) + Maths.WAD);
```

(A), assets and a part to be written down in `2. settle debt with pool reserves` calculation are rounded down with `floorWmul` and `floorWdiv`:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L133-L146

```solidity
        if (borrower.t0Debt != 0 && borrower.collateral == 0) {
            // 2. settle debt with pool reserves
            uint256 assets = Maths.floorWmul(poolState_.t0Debt - result_.t0DebtSettled + borrower.t0Debt, poolState_.inflator) + params_.poolBalance;

            uint256 liabilities =
                // require 1.0 + 1e-9 deposit buffer (extra margin) for deposits
                Maths.wmul(DEPOSIT_BUFFER, Deposits.treeSum(deposits_)) +
                auctions_.totalBondEscrowed +
                reserveAuction_.unclaimed;

            // settle debt from reserves (assets - liabilities) if reserves positive, round reserves down however
            if (assets > liabilities) {
                borrower.t0Debt -= Maths.min(borrower.t0Debt, Maths.floorWdiv(assets - liabilities, poolState_.inflator));
            }
```

(7), when there are no deposits left settlement puts all the collateral into lowest 7388 bucket, not clearing the remaining debt:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L410-L421

```solidity
            } else {
                // Deposits in the tree is zero, insert entire collateral into lowest bucket 7388
                Buckets.addCollateral(
                    buckets_[vars.index],
                    params_.borrower,
                    0,  // zero deposit in bucket
                    remainingCollateral_,
                    vars.price
                );
                // entire collateral added into bucket, no borrower pledged collateral remaining
                remainingCollateral_ = 0;
            }
```

(8), if pool's balance is zero (100 units of quote token added by depositors, the same removed by the borrower, and the kicker bond was added), then, ignoring reserves part as being small after the successful auction, `assets = 5 + totalBondEscrowed`, `liabilities = totalBondEscrowed`, so around `5` is what will be left for the `borrower.t0Debt` after step 2 of the settlement:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L133-L146

```solidity
        if (borrower.t0Debt != 0 && borrower.collateral == 0) {
            // 2. settle debt with pool reserves
            uint256 assets = Maths.floorWmul(poolState_.t0Debt - result_.t0DebtSettled + borrower.t0Debt, poolState_.inflator) + params_.poolBalance;

            uint256 liabilities =
                // require 1.0 + 1e-9 deposit buffer (extra margin) for deposits
                Maths.wmul(DEPOSIT_BUFFER, Deposits.treeSum(deposits_)) +
                auctions_.totalBondEscrowed +
                reserveAuction_.unclaimed;

            // settle debt from reserves (assets - liabilities) if reserves positive, round reserves down however
            if (assets > liabilities) {
                borrower.t0Debt -= Maths.min(borrower.t0Debt, Maths.floorWdiv(assets - liabilities, poolState_.inflator));
            }
```

Then step 3 of the settlement, `_forgiveBadDebt`, will not change anything as there are no deposits left:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L148-L157

```solidity
            // 3. forgive bad debt from next HPB
            if (borrower.t0Debt != 0) {
                borrower.t0Debt = _forgiveBadDebt(
                    buckets_,
                    deposits_,
                    params_,
                    borrower,
                    poolState_.inflator
                );
            }
```

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L442-L456

```solidity
    function _forgiveBadDebt(
        mapping(uint256 => Bucket) storage buckets_,
        DepositsState storage deposits_,
        SettleParams memory params_,
        Borrower memory borrower_,
        uint256 inflator_
    ) internal returns (uint256 remainingt0Debt_) {
        remainingt0Debt_ = borrower_.t0Debt;

        // loop through remaining buckets if there's still debt to forgive
        while (params_.bucketDepth != 0 && remainingt0Debt_ != 0) {

            (uint256 index, , uint256 scale) = Deposits.findIndexAndSumOfSum(deposits_, 1);
>>          uint256 unscaledDeposit          = Deposits.unscaledValueAt(deposits_, index);
>>          uint256 depositToRemove          = Maths.wmul(scale, unscaledDeposit);
```

So the resulting debt will remain positive and `_settleAuction() -> _removeAuction()` will not be called:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/SettlerActions.sol#L168-L178

```solidity
        // if entire debt was settled then settle auction
        if (borrower.t0Debt == 0) {
            (borrower.collateral, ) = _settleAuction(
                auctions_,
                buckets_,
                deposits_,
                params_.borrower,
                borrower.collateral,
                poolState_.poolType
            );
        }
```

## Tool used

Manual Review

## Recommendation

Consider adding an additional writing off mechanics for this case, so the pool can be made unstuck without the need of any material investment, so it can be done quickly and by any independent actor.

The investment itself doesn't look to be needed as no parties with accounted interest are left in the pool at that point: everything was cleared and the remaning part is of technical nature, it can be decsribed as a result of too much reserves being sold out via regular reserve auction. That's the another version for a solution, a `debt - deposits` can be guarded in the reserves as a stability part needed for clearance.


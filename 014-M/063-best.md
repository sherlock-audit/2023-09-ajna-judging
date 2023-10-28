Restless White Bee

high

# Settlement can reach the state when writing off the debt exhausts all deposits and current reserves, freezing the pool with a not cleared auction
## Summary

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
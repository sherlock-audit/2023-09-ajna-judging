Bumpy Punch Albatross

high

# First pool borrower pays extra interest
## Summary

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

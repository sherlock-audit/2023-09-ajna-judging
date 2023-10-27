Bumpy Punch Albatross

medium

# lenderKick incorrectly sets LUP
## Summary

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

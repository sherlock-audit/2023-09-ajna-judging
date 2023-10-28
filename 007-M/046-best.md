Restless White Bee

medium

# Unsafe truncation casting is used for a number of state variables, including uncapped ones
## Summary

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
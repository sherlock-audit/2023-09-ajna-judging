Restless White Bee

medium

# repayDebt collateral check logic in ERC-721 case is more loose than _isCollateralized based one used across the protocol
## Summary

If NFT collateral becomes fractional for any reason ERC721Pool borrowers might disable taking by creating positions having collateral less than `1` NFT.

## Vulnerability Detail

If NFT collateral becomes fractional then it can be possible, given the necessary collateral price and debt levels, to remove ERC-721 collateral so that remainder is less than `1`, say `1` can be removed from `1.9` to create a `0.9` collateral in a position. Such loan will not be liquidable as this amount of collateral cannot be removed from the position.

I.e. in general it is now possible to create a loan with `_isCollateralized(debt, collateral, lup, pool.poolType()) == false` with pulling collateral via `repayDebt()`.

## Impact

While such positions are healthy as of `repayDebt()`, the fact that they cannot be liquidated means that given enough number of such positions worsening market conditions can drive the pool towards unusable state as his health cannot be sustained.

Since at the moment there looks to be no way to orchestrate the fractional collateral position outside the auction, while `repayDebt()` will not work while a loan is in auction, the possibility of some not yet discovered way to facilitate fractional collateral (which is otherwise valid from the viewpoint of the system) cannot be fully ruled out, so deeming this issue to have high impact and low probability, placing the overall severity to be medium.

## Code Snippet

ERC721Pool's `repayDebt()` only ensures that requested amount of collateral is a full wad:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/ERC721Pool.sol#L215-L237

```solidity
    function repayDebt(
        ...
    ) external nonReentrant returns (uint256 amountRepaid_) {
        PoolState memory poolState = _accruePoolInterest();

        // ensure accounting is performed using the appropriate token scale
        if (maxQuoteTokenAmountToRepay_ != type(uint256).max)
            maxQuoteTokenAmountToRepay_ = _roundToScale(maxQuoteTokenAmountToRepay_, poolState.quoteTokenScale);

        RepayDebtResult memory result = BorrowerActions.repayDebt(
            auctions,
            deposits,
            loans,
            poolState,
            borrowerAddress_,
            maxQuoteTokenAmountToRepay_,
>>          Maths.wad(noOfNFTsToPull_),
            limitIndex_
        );
```

But doesn't check that remaining collateral can be extracted:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/external/BorrowerActions.sol#L287-L320

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
>>              borrower.collateral - encumberedCollateral < collateralAmountToPull_
            ) revert InsufficientCollateral();

            // stamp borrower Np to Tp ratio when pull collateral action
            vars.stampNpTpRatio = true;

            borrower.collateral -= collateralAmountToPull_;

            result_.poolCollateral -= collateralAmountToPull_;
        }

        // check limit price and revert if price dropped below
        _revertIfPriceDroppedBelowLimit(result_.newLup, limitIndex_);

        // update loan state
        Loans.update(
            ...
        );
```

While `_isCollateralized()` uses floor amount (i.e. the extractable part only) of collateral NFTs to check for a given loan collaterization:

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/helpers/PoolHelper.sol#L158-L170

```solidity
    function _isCollateralized(
        ...
    ) pure returns (bool) {
        if (type_ == uint8(PoolType.ERC20)) return Maths.wmul(collateral_, price_) >= debt_;
        else {
            //slither-disable-next-line divide-before-multiply
>>          collateral_ = (collateral_ / Maths.WAD) * Maths.WAD; // use collateral floor
>>          return Maths.wmul(collateral_, price_) >= debt_;
        }
    }
```

## Tool used

Manual Review

## Recommendation

Consider unifying the logic and using `_isCollateralized()` version of check in pulling logic of `repayDebt()`.
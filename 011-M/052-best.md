Restless White Bee

medium

# settlePoolDebt places the collateral to the lowest bucket, allowing for stealing it with back-running
## Summary

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
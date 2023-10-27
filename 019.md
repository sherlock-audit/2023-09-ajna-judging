Real Pastel Starfish

medium

# Price curve for Current Reserve Auction not updated
## Summary

Since the Ajna update, the price curve of the Dutch auction was changed from a linear function to a new function that allows a more efficient discovery of the price, this change applies to liquidation auctions and to reserves auction. While the liquidation auction has been correctly updated to follow the new function, the reserves auction is not updated and still follows the old linear function.

## Vulnerability Detail

In the new whitepaper (section **9.1 Buy and Burn Mechanism**) it's stated that:

> The price of the auction decays towards 0 starting with 6 twenty minute halvings, followed by 6 two hour halvings, followed by hour halvings till the end of the 72 hour auction.

If we look at the code, we see that the function that calculated the auction price for reserves is the following:
https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/helpers/PoolHelper.sol#L331-L341
```solidity
function _reserveAuctionPrice(
    uint256 reserveAuctionKicked_
) view returns (uint256 price_) {
    if (reserveAuctionKicked_ != 0) {
        uint256 secondsElapsed   = block.timestamp - reserveAuctionKicked_;
        uint256 hoursComponent   = 1e27 >> secondsElapsed / 3600;
        uint256 minutesComponent = Maths.rpow(MINUTE_HALF_LIFE, secondsElapsed % 3600 / 60);

        price_ = Maths.rayToWad(1_000_000_000 * Maths.rmul(hoursComponent, minutesComponent));
    }
}
```

The price curve in the above function decays exponentially with a half life of 1-hour, and it doesn't adhere to the specification in the whitepaper. 

## Impact

The price curve of the reserves auction doesn't follow the specification on the whitepaper so that will cause a more inefficient price discovery, as well as misleading all actors in the auction causing a chaotic and unfair auction. 

## Code Snippet

https://github.com/sherlock-audit/2023-09-ajna/blob/main/ajna-core/src/libraries/helpers/PoolHelper.sol#L331-L341

## Tool used

Manual Review

## Recommendation

Consider adapting the `_reserveAuctionPrice` function to match the specification. 


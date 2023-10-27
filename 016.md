Skinny Frost Perch

medium

# Wrong auctionPrice used in calculating BPF to determine bond reward (or penalty)
## Summary
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

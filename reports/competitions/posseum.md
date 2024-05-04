
# High

## Full Title 
`AdapterV1.sol#addLiquidity()` - The contract always uses the entire balance of PSM of the Adapter, which can lead to weak slippage. 

## Description 
When `setPortalEnergy` is called with `mode = 1` the protocol will first swap PSM for WETH using 1inch, then use PSM and WETH to add liquidity to a Ramses pair.

In order to do this, the user has to specify 2 slippages, 1 for the 1inch swap and 1 when calculating how much liquidity needs to be added to the Ramses pair.

Let's go over the code one step at a time.

First, we have to sell our portal tokens to get PSM which is sent to the adapter.
Second, we do a swap using 1inch which will swap some amount of the received PSM for WETH, when the swap is done the WETH is received in the adapter.
Third, we get PSMBalance and WETHBalance which are then passed to `_addLiquidity` which will calculate `amountPSM` and `amountWETH` which need to be transferred to the Ramses pair.

1inch has a case, where the tokens that were used for the swap were actually more than necessary, so the unspent amount is refunded to `msg.sender`. This is what this part of the code is used for.

## Impact
Inflated/unfair slippage used for user

## Attack Scenario
To better visualize this, I'll give an example. The correlation between PSM and WETH is simplified to better understand the issue.

1. Alice wants to sell her portal tokens and then add liquidity in the Ramses pair.
2. She calls sellPortalTokens.
3. She sells 200 PSM worth of portal tokens and she wants to use 100 PSM to swap for 1 WETH, the other 100 she wants to use to add liquidity to the pool.
4. 1inch uses up only 50 PSM for 1 WETH, so the other 50 PSM are refunded directly to the Adapter.
5. PSMBalance is now 150, as the code uses the entire balance of the Adapter to calculate how much liquidity needs to be added.
6. Alice specified 90 PSM as minPSM, as she expected only 100 PSM to be used to add liquidity, but now 150 is used.
7. Due to market conditions, amountPSM = 100, but PSMBalance is 150, meaning that the price dropped a lot more than Alice would have wanted.
8. Because Alice specified minPSM = 90 the tx goes through and Alice let a tx which slipped in price a lot more than she would have agreed to, if she knew that the 150 PSM would be used to add liquidity instead of 100 PSM.

## Recommendation
It's best to always refund any unused PSM from the 1Inch swap, this will make the function much more predictable and won't result in cases where a tx goes through with a much more inflated PSM amount, thus making the user's slippage inflated and very high.

# Low

## Full Title 
`AdapterV1.sol#swapOneInch()` - Remaining tokens should be sent to `_swap.recevier` instead of `msg.sender`

## Description 

When calling `swapOneInch` we specify swapData and `_forLiquidity`. Inside `swapData` we specify a receiver who will receive the tokens.

There is a case when if `_forLiquidity = false` we check if there are any remaining PSM tokens by subtracting `spentAmount_` from `_swap.psmAmount`.

If there are any remaining tokens, they are sent to `msg.sender`. Considering we are specifying a `swapData.receiver`, if there are any remaining PSM tokens, they should be sent to `swapData.receiver`.

## Impact 
Inconvenience and potential gas costs, as `msg.sender` has to transfer the tokens to `swapData.receiver`.

## Recommendation
Rework the code, to the following:
```
            if (remainAmount > 0) PSM.safeTransfer(_swap.recevier, remainAmount);
```



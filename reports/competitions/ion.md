# Medium

## Description
When a user wants to withdraw his stakings, he return his shares by calling `withdraw` or `redeem`, those two functions calls `_withdrawFromIonPool`, which
s then iterating over `withdrawQueue` from first added pool. The problem here is that if a pool is paused, the tx will revert here, because `pool.withdraw` has `whenNotPaused` modifier and user
won't be able to receive his funds, even if there is available liquidity in next pools in the queue.

## Impact
Impact is temporary DoS.

## Recommendation

Check if the pool is paused and if it is, skip withdrawing from it

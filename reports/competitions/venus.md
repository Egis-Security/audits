## Severity
Medium

## Full title

`OmnichainGovernanceExecutor.sol#_blockingLzReceive()` - If the contract gets paused, it will block cross chain communication because of where the modifier is placed

## Description
`OmnichainGovernanceExecutor` implements `Pausable`, through `BaseOmnichainControllerDest`

The modifier `_whenNotPaused` is attached to `_blockingLzReceive`

`_blockingLzReceive` is hit and the code will execute in the following order.

whenNotPaused -> _blockingLzReceive -> _nonblockingLzReceive

This means that if the contract is paused, it will revert outside `_noblockingLzReceive` and the tx will not be stored inside `failedMessages` to be later retried with `retryMessage`, instead it will block the communicaton channel between the two chains.

If any subsequent messages are sent to the destination chain, while the cross chain communication is blocked, then the nonces will become out of sync, permanently blocking the channel between the two chains.

Another scenario that can occur, with technically the same root cause:

`paused = false`
Proposal1 fails inside `_nonblockingLzReceive` and is stored inside `failedMessages`
`paused = true`
Someone calls `retryMessage` and successfully retries the message and `_nonblockingLzReceive` is hit and the tx doesn't revert, because `whenNotPaused` isn't attached to the function.
## Recommendation
We recommend adding the `_whenNotPaused` modifier to `_nonblockingLzReceive` instead of `_blockingLzReceive`. This way if the destination is paused, the messages will be stored inside `failedMessages` where they can later be executed, the cross chain communication will not be blocked and users won't have to create completely new proposals on the source chain.

This will also stop the second scenario from occurring when the destination gets paused.
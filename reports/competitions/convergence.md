# Medium
## User looses StakeDao rewards, if he misses to call claimCvgCvxRewards for cycle
Description:
Description\

Inside StakingServiceBase::_claimCvgCvxRewards we set maxLengthRewards to the cycle, when was the last user interaction with contract, which is not correct, because we cannot be sure when will position owner claim his rewards. The impact is that user looses rewards, which are stucked in the contract

Attack Scenario
Imagine the following scenario:
After cycle 0, on processCvxRewards is called and in this moment CvxAssetStakerBuffer::pullReward set only token "A" as a reward token related to Stake Dao

            _cvxRewardsByCycle[_cvgStakingCycle][erc20Id] = ICommonStruct.TokenAmount({
                token: _token,
                amount: _cvxRewardsByCycle[_cvgStakingCycle][erc20Id].amount + _rewardAssets[i].amount
            });
So token A erc20Id is set as 1.

After cycles 2 and 3, we have respectively new reward tokens B and C, which are set as 2 and 3 inside _tokenToId[_token] mapping.

The problem now is that if Bob misses to call claimCvgCvxRewards for a couple of cycles, he would miss to receive those tokens and they are stuck in the contract, because Bob's stakings are taken into consideration while calculating each user reward share.

If Bob stakes at cycle = 0,
5 cycles have passed and he calls claimCvgCvxRewards:
nextClaimableCvx = 1 (nextCycle when he has staked)
uint256 maxLengthRewards = 1(nextClaimableCvx);
ICommonStruct.TokenAmount[] memory _totalRewardsClaimable = new ICommonStruct.TokenAmount[](1(maxLengthRewards));
Inisde StakeDAO part we will only handle first reward token:
_cvxRewardsByCycle[nextClaimableCvx][1] (which is token A) for all missed cycles, while there are more erc20ids ,which holds rewards for bob inside _cvxRewardsByCycle[nextClaimableCvx][erc20Id + 1], but we won't handle them, because the for loop iterates only for the first erc20id (because we have set maxLengthRewards to the first cycle bob has staked)

At the end of the function we set _nextClaims[tokenId].nextClaimableCvx = 5(actualCycle); , which means that we have processed all pending rewards for all previous cycles, which is not true.

For comparison, if user calls claimCvgCvxRewards for each cycle, he would receive more of his rewards, because maxRewardLength is obtained from here, but even in this case, some rewards may be missed.

# Low

## CvxConvergenceLocker::sentTokens doesn't protect from newly added reward tokens
### Description
Inside CvxConvergenceLocker there is restricted function sendTokens, which has modifier onlyOwner and it is used to transfer locked funds in the contract, which are different from the reward tokens as we can see the following lines:

     require(address(tokens[i]) != address(CVX), "CVX_CANNOT_BE_TRANSFERRED");
            require(address(tokens[i]) != address(CRV), "CRV_CANNOT_BE_TRANSFERRED");
            require(address(tokens[i]) != address(FXS), "FXS_CANNOT_BE_TRANSFERRED");
            require(address(tokens[i]) != address(this), "CVGCVX_CANNOT_BE_TRANSFERRED");

The problem is that new reward tokens may be added from CVX_LOCKER using addReward, but sendTokens doesn't check for such new reward tokens and so owner may tranfer them out of the locker contract.

### Attack Scenario

Owner of CvxLocker calls addReward with a new reward token different from [CVX, CRV, FXS] (maybe CVX1)

Some reward value is accumulated for CvxConvergenceLocker and when CvxConvergenceLocker::pullRewards, it is transferred to the locker contract, but CVX1 is still not added to rewardTokensConfiguration, so it is left in this contract

Now owner calls sendTokens with the address of CVX1 and user rewards are sent outside of the contract.
Attachments


### Mitigation
Inside sendTokens check dynamically wheter tokens[i] is not present in CvxLocker::rewardTokens()

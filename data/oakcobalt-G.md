### Gas-01 Wasteful operation in the staking flow when `unlockTime` doesn't change
**Instances(2)**
**Total Gas Saved(10000)**
In `selfStake()` and `communityStake()`, a user can stake an additional amount with a future `unlockTime`. 

However, when `unlockTime` doesn't change from the old lock, wasteful state operation is performed - the same unlock time is re-written to `selfStakes` or `communityStakes`.
(1)
```solidity
//id-staking-v2/contracts/IdentityStaking.sol
  function selfStake(uint88 amount, uint64 duration) external whenNotPaused {
  ...
      if (
      // Must be between 12 weeks and 104 weeks
      unlockTime < block.timestamp + 12 weeks ||
      unlockTime > block.timestamp + 104 weeks ||
      // Must be later than any existing lock
      unlockTime < selfStakes[msg.sender].unlockTime
    ) {
      revert InvalidLockTime();
    }

    selfStakes[msg.sender].amount += amount;
    //@audit Gas: wasteful operation when unlockTime == existing lock unlockTime
|>    selfStakes[msg.sender].unlockTime = unlockTime;

```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L247)
(2)

```solidity
  function communityStake(address stakee, uint88 amount, uint64 duration) external whenNotPaused {
  ...
      //@audit Gas: wasteful operation when unlockTime == existing lock unlockTime
    communityStakes[msg.sender][stakee].amount += amount;
|>    communityStakes[msg.sender][stakee].unlockTime = unlockTime;
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L338)

Note: Editing the existing value of a storage slot costs 5000 gas.

Recommendations:
Check `unlockTime` and only write to storage when it differs from the old `unlockTime`.

### Gas-02 emit outside of for-loop to save gas (Not included in bot report)
**Instances(2)**
**Total Gas Saved(Various)**
In `slash()`, `Slash` events are emitted inside for-loops. 

Each event emitting has an overhead of 375 gas. Consider emit the event outside of for-loops.
(1)
```solidity
  function slash(
    address[] calldata selfStakers,
    address[] calldata communityStakers,
    address[] calldata communityStakees,
    uint88 percent
  ) external onlyRole(SLASHER_ROLE) whenNotPaused {
...
      for (uint256 i = 0; i < numSelfStakers; i++) {
      ...
|>            emit Slash(staker, slashedAmount, currentSlashRound);
      }
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L467)
(2)
```solidity
  function slash(
    address[] calldata selfStakers,
    address[] calldata communityStakers,
    address[] calldata communityStakees,
    uint88 percent
  ) external onlyRole(SLASHER_ROLE) whenNotPaused {
...
          for (uint256 i = 0; i < numCommunityStakers; i++) {
         ...
         
 |>              emit Slash(staker, slashedAmount, currentSlashRound);
    }
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L497)

Recommendations:
emit events outside of for-loops.

### Gas-03 When `slashedAmount` rounds to 0, wasteful operation in `slash()`
**Instances(2)**
**Total Gas Saved(Various)**
In `slash()`, It's possible that `slashedAmount` might round down to 0. In this case, wasteful zero value math and storage update will be performed.

Note that each storage slot edit costs 5000 gas. 

(1)
```solidity
  function slash(
    address[] calldata selfStakers,
    address[] calldata communityStakers,
    address[] calldata communityStakees,
    uint88 percent
  ) external onlyRole(SLASHER_ROLE) whenNotPaused {
...
      uint88 slashedAmount = (percent * selfStakes[staker].amount) / 100;
...
      totalSlashed[currentSlashRound] += slashedAmount;
...
      sStake.slashedAmount += slashedAmount;
      sStake.amount -= slashedAmount;
      userTotalStaked[staker] -= slashedAmount;
...
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L462-L465)

(2)
```solidity
...
      comStake.slashedAmount += slashedAmount;
      comStake.amount -= slashedAmount;

      userTotalStaked[staker] -= slashedAmount;
...
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L492-L495)

Recommendations:
check `slashedAmount` and only update storage when `slahsedAmount` is non-zero.
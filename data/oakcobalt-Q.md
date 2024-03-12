### Low-01 Locks can be extended to an old `unlockTime`
**Instances(2)**
In IdentityStaking.sol, self-stake or community-stake lock duration can be extended. However, checks are insufficient and allow a new lock to have the same `unlockTime` as the old lock.
(1)
```solidity
//id-staking-v2/contracts/IdentityStaking.sol
  function extendSelfStake(uint64 duration) external whenNotPaused {
...
    uint64 unlockTime = duration + uint64(block.timestamp);

    if (
      // Must be between 12 weeks and 104 weeks
      unlockTime < block.timestamp + 12 weeks ||
      unlockTime > block.timestamp + 104 weeks ||
      // Must be later than any existing lock
|>      unlockTime < selfStakes[msg.sender].unlockTime //@audit This allows the new unlockTime to be the same as the old unlockTime.
    ) {
      revert InvalidLockTime();
    }

    selfStakes[msg.sender].unlockTime = unlockTime;
...
}
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L273)
(2)
```solidity
  function extendCommunityStake(address stakee, uint64 duration) external whenNotPaused {
...
    if (
      // Must be between 12 weeks and 104 weeks
      unlockTime < block.timestamp + 12 weeks ||
      unlockTime > block.timestamp + 104 weeks ||
      // Must be later than any existing lock
|>      unlockTime < comStake.unlockTime. //@audit This allows the new unlockTime to be the same as the old unlockTime.
    ) {
      revert InvalidLockTime();
    }
...
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L371)

Recommendations:
Change to `unlockTime <= selfStakes[msg.sender].unlockTime` to ensure new locktime will be different.

### Low-02 Redundancy in code comments
**Instances(2)**
In IdentityStaking.sol, there are (2) cases of code comment redundancy. In `communityStake()` and `extendCommunityStake()`, change `12-104 weeks and 104 weeks` into `12-104 weeks`.
(1)
```solidity
|>  /// @dev The duration must be between 12-104 weeks and 104 weeks, and after any existing lock for this staker+stakee
  function communityStake(address stakee, uint88 amount, uint64 duration) external whenNotPaused {
...
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L310)
(2)
```solidity
|>  /// @dev The duration must be between 12-104 weeks and 104 weeks, and after any existing lock for this staker+stakee
  ///      The unlock time is calculated as `block.timestamp + duration`
  function extendCommunityStake(address stakee, uint64 duration) external whenNotPaused {
...
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L332)

Recommendations:
Change `12-104 weeks and 104 weeks` into `12-104 weeks`.

### Low-03 Dust staking for self or community is allowed, which might cause slash to be more expensive to maintain over time 
**Instances(2)**
A user can stake dust amount for both self or other community members. If the user commits offenses, the intended behavior is `if I have staked GTC on other people, and I have misbehaved, then all my stakes on others will be slashed`. And if the stakee commits offsenses, then `those particular stakes will be slashed`.

Both scenario means that the staker->stakee amounts will be slashed regardless of whether the staker or the stakee commits offenses. When staker takes dusts amount on multiple stakees, or a stakee has many dust amount staked by other, this will make the required `slash()` flows more expansive due to dust value updates.
(1)
```solidity
  function selfStake(uint88 amount, uint64 duration) external whenNotPaused {
...
    if (amount == 0) {
      revert AmountMustBeGreaterThanZero();
    }
...}
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L230-L231)
(2)
```solidity
  function communityStake(address stakee, uint88 amount, uint64 duration) external whenNotPaused {
...
    if (amount == 0) {
      revert AmountMustBeGreaterThanZero();
    }
...
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L321-L322)
Recommendations:
Prevent dust amount staking

### Low-04 `slash()` is vulnerable to `lockAndBurn()` racing, accounting of consecutive slashes might be inconsistent
**Instances(1)**
In normal circumstances, consecutive slash amounts will be rolled over. For example, Alice is slashed in round 1. And if Alice is slashed again in round 2, the slash amount from round 1 will be rolled over in round 2. See [doc](https://github.com/code-423n4/2024-03-gitcoin/blob/main/id-staking-v2/README.md#appendix-b-slashing-in-consecutive-rounds).

However, the above behavior cannot be guaranteed due to a possible permissionless `lockAndBurn()` racing. 
When `slash()` on round 2 is settled before `lockAndBurn()`. The doc-described behavior is preserved.  
```solidity
  function slash(
    address[] calldata selfStakers,
    address[] calldata communityStakers,
    address[] calldata communityStakees,
    uint88 percent
  ) external onlyRole(SLASHER_ROLE) whenNotPaused {
...
      if (sStake.slashedInRound != 0 && sStake.slashedInRound != currentSlashRound) {
          //@audit this if body will run when `slash()` settles before `lockAndBurn()`
|>        if (sStake.slashedInRound == currentSlashRound - 1) {
          // If this is a slash from the previous round (not yet burned), move
          // it to the current round
          totalSlashed[currentSlashRound - 1] -= sStake.slashedAmount;
          totalSlashed[currentSlashRound] += sStake.slashedAmount;
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L448)

But if `lockAndBurn()` settles first, the staked amount of `sStake.slashedInRound` from round 1 will be burned, and the slash amount from round 2 will be counted in round 3 even though the `slash()` is submitted in round 2.
```solidity
  function slash(
    address[] calldata selfStakers,
    address[] calldata communityStakers,
    address[] calldata communityStakees,
    uint88 percent
  ) external onlyRole(SLASHER_ROLE) whenNotPaused {
...
      if (sStake.slashedInRound != 0 && sStake.slashedInRound != currentSlashRound) {
        if (sStake.slashedInRound == currentSlashRound - 1) {
...
        } else {
          //@audit else body will run if lockAndburn settles before slash()
          // Otherwise, this is a stale slash and can be overwritten
|>          sStake.slashedAmount = 0;
        }
...
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L455)

As seen above, `slash()` submitted close to the end of a round(90 days `burnRoundMinimumDuration`), might be counted either at the next round or current round due to potential `lockAndBurn` racing.  When `slash()` settles at the next round, the user's previous round slash amount will not be rolled over, and the slash amount will be directly burned instead of being preserved for the next round.

Recommendations:
Consider having `lockAndBurn()` access-controlled by the protocol to ensure `slash()`,and `lockAndBurn()` settle in the correct order.

### Low-05  Racing conditions between `release()` and `slash()`, might cause `release()` tx reverts
**Instances(1)**
Suppose a user is approved for a refund of fines from a previous round. But before `release()` is settled, the user is slashed again in the current round. `release()` and `slash()` racing might occur and result in uncertain behavior: (1) Either `release()` succeeds, the user is first refunded with previous slashed amount and then slashed with a new fine; (2) Or `release()` will revert, user's previous and current slashed amount are combined. The user doesn't receive a refund.

```solidity
//id-staking-v2/contracts/IdentityStaking.sol
  function release(
    address staker,
    address stakee,
    uint88 amountToRelease,
    uint16 slashRound
  ) external onlyRole(RELEASER_ROLE) whenNotPaused {
...
    if (staker == stakee) {
...
        //@audit if slash() settles first, slashedInRound will be updated to the current round. release() is submitted for previous round. This cause release() to revert.
|>      if (selfStakes[staker].slashedInRound != slashRound) {
        revert FundsNotAvailableToReleaseFromRound();
      }
...
    } else {
...
        //@audit if slash() settles first, slashedInRound will be updated to the current round. release() is submitted for previous round. This cause release() to revert.
|>      if (communityStakes[staker][stakee].slashedInRound != slashRound) {
        revert FundsNotAvailableToReleaseFromRound();
      }
...
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L558)

`release()` tx might revert due to `slash()` racing. 

Recommendations:
In this case, `RELEASER_ROLE` and `SLASHER_ROLE` need to be coordinated to prevent random reverts. 

### Low-06 Distinguish event emitting between slashing self-stakes and slashing community-stakes
**Instances(1)**
In `slash()`, the event emitted for slashing of self-stakes and slashing of community-stakes have the same fields. From an off-chain perspective, when a staker is slashed for both self-stakes and community-stakes, `commnuityStakee` address will not be emitted, and it might be hard to distinguish between self-stakes and community-stakes from the same tx.

```solidity
//id-staking-v2/contracts/IdentityStaking.sol

  function slash(
    address[] calldata selfStakers,
    address[] calldata communityStakers,
    address[] calldata communityStakees,
    uint88 percent
  ) external onlyRole(SLASHER_ROLE) whenNotPaused {
...
    for (uint256 i = 0; i < numSelfStakers; i++) {
...
|>      emit Slash(staker, slashedAmount, currentSlashRound);
    }
...
    for (uint256 i = 0; i < numCommunityStakers; i++) {
...
|>      emit Slash(staker, slashedAmount, currentSlashRound);
    }
  }
```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L497)

Recommendations:
consider adding `stakee` indexed field in the event of community-stake slashing.


### Low-07 A user might be slashed with 0 amount fine due to rounding
**Instances(1)**

When a user stake dust amount, `slash()` might round the total `slashedAmount` to 0, resulting in 0 amount slashing.

```solidity
//id-staking-v2/contracts/IdentityStaking.sol
  function slash(
    address[] calldata selfStakers,
    address[] calldata communityStakers,
    address[] calldata communityStakees,
    uint88 percent
  ) external onlyRole(SLASHER_ROLE) whenNotPaused {
...
      for (uint256 i = 0; i < numSelfStakers; i++) {
      ...
            //@audit when dust amount, slashedAmount might round to 0, slasher role only input a percentage, the actual slashedAmount, should factor in rounding correctly.
|>      uint88 slashedAmount = (percent * selfStakes[staker].amount) / 100;

```
(https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L443)

Recommendations:
Disallow dust amount staking. 

### Low-08 Consider disabling initialize in the implementation contract
**Instances(1)**
id-staking-v2/contracts/IdentityStaking.sol is an implementation contract. Consider disabling the option of initializing the implementation itself by calling `_disableInitializers()` in constructor as recommended by openzeppelin.

```solidity
//id-staking-v2/node_modules/@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol
    /**
     * @dev Locks the contract, preventing any future reinitialization. This cannot be part of an initializer call.
     * Calling this in the constructor of a contract will prevent that contract from being initialized or reinitialized
     * to any version. It is recommended to use this to lock implementation contracts that are designed to be called
     * through proxies.
     *
     * Emits an {Initialized} event the first time it is successfully executed.
     */
    function _disableInitializers() internal virtual {
```

Recommendations:
Implement `_disableInitializers()` in id-staking-v2/contracts/IdentityStaking.sol.
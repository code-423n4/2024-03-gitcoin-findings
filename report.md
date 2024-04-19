---
sponsor: "Gitcoin Passport"
slug: "2024-03-gitcoin"
date: "2024-04-19"
title: "Gitcoin Passport - Identity Staking Invitational"
findings: "https://github.com/code-423n4/2024-03-gitcoin-findings/issues"
contest: 337
---

# Overview

## About C4

Code4rena (C4) is an open organization consisting of security researchers, auditors, developers, and individuals with domain expertise in smart contracts.

A C4 audit is an event in which community participants, referred to as Wardens, review, audit, or analyze smart contract logic in exchange for a bounty provided by sponsoring projects.

During the audit outlined in this document, C4 conducted an analysis of the Gitcoin Passport Identity Staking smart contract system written in Solidity. The audit took place between March 6 — March 12, 2024.

Following the C4 audit, 3 wardens ([oakcobalt](https://code4rena.com/@oakcobalt), [0xDING99YA](https://code4rena.com/@0xDING99YA) and [Stormy](https://code4rena.com/@Stormy)) reviewed the mitigations for all identified issues; the [mitigation review report](#mitigation-review) is appended below the audit report.

## Wardens

In Code4rena's Invitational audits, the competition is limited to a small group of wardens; for this audit, 4 wardens contributed reports to Gitcoin Passport Identity Staking:

  1. [oakcobalt](https://code4rena.com/@oakcobalt)
  2. [Stormy](https://code4rena.com/@Stormy)
  3. [0xDING99YA](https://code4rena.com/@0xDING99YA)
  4. [EV\_om](https://code4rena.com/@EV_om)

This audit was judged by [Alex the Entreprenerd](https://code4rena.com/@GalloDaSballo).

Final report assembled by [thebrittfactor](https://twitter.com/brittfactorC4).

# Summary

The C4 analysis yielded an aggregated total of 1 unique vulnerability. Of these vulnerabilities, 1 received a risk rating in the category of HIGH severity and 0 received a risk rating in the category of MEDIUM severity.

Additionally, C4 analysis included 2 reports detailing issues with a risk rating of LOW severity or non-critical. There were also 1 reports recommending gas optimizations.

All of the issues presented here are linked back to their original finding.

# Scope

The code under review can be found within the [C4 Gitcoin Passport Identity Staking repository](https://github.com/code-423n4/2024-03-gitcoin), and is composed of 2 smart contracts written in the Solidity programming language and includes 300 lines of Solidity code.

In addition to the known issues identified by the project team, an [Automated Findings report](https://github.com/code-423n4/2024-03-gitcoin/blob/main/4naly3er-report.md) was generated using the [4naly3er bot](https://github.com/Picodes/4naly3er) and all findings therein were classified as out of scope.

# Severity Criteria

C4 assesses the severity of disclosed vulnerabilities based on three primary risk categories: high, medium, and low/non-critical.

High-level considerations for vulnerabilities span the following key areas when conducting assessments:

- Malicious Input Handling
- Escalation of privileges
- Arithmetic
- Gas use

For more information regarding the severity criteria referenced throughout the submission review process, please refer to the documentation provided on [the C4 website](https://code4rena.com), specifically our section on [Severity Categorization](https://docs.code4rena.com/awarding/judging-criteria/severity-categorization).

# High Risk Findings (1)
## [[H-01] `userTotalStaked` invariant will be broken due to vulnerable implementations in `release()`](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/9)
*Submitted by [oakcobalt](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/9), also found by [oakcobalt](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/10), [Stormy](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/17), and [0xDING99YA](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/2)*

`userTotalStaked` invariant will be broken due to vulnerable implementations in `release()`. Users might lose funds due to underflow errors in withdraw methods.

### Proof of Concept

According to the [readme](https://github.com/code-423n4/2024-03-gitcoin/tree/main), `userTotalStaked` invariant should always hold true:

`userTotalStaked[address] = selfStakes[address].amount + sum(communityStakes[address][x].amount for all x staked on by this address)`

However, this can be broken in `release()` flow due to `userTotalStaked` is not updated together with `selfStakes[address].amount` or `communityStakes[address][x].amount`.

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
      selfStakes[staker].slashedAmount -= amountToRelease;
      //@audit selfStakes[staker].amount is updated but `userTotalStaked` is not
|>    selfStakes[staker].amount += amountToRelease;
    } else {
...
      communityStakes[staker][stakee].slashedAmount -= amountToRelease;
      //@audit communityStakes[staker].amount is updated but `userTotalStaked` is not
|>    communityStakes[staker][stakee].amount += amountToRelease;
    }
...
```

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L562-L563

For comparison, in other flows such as staking, withdrawing and slashing, `userTotalStaked` is always updated in sync with `selfStakes`/`communityStakes`.

POC:

1. Alice `selfStakes()` 100000 ether and `communityStakes()` 100000 at round 1.
2. Alice's `selfStake` and `communityStake` are slashed by 80% each.
3. Alice appealed and was released the full slashed amount. Alice's staked balance is restored to 100000 ether each. But `userTotalStaked` is not restored.
4. Alice's unlocked but cannot withdraw the 100000x2 ether balance due to underflow. She can only withdraw 20000x2 ether.

See test below:

In `id-staking-v2/test/IdentityStaking.ts`, first add `import { PANIC_CODES} from "@nomicfoundation/hardhat-chai-matchers/panic";`. 

Then copy this test inside `describe("slashing/releasing/burning tests", function () {`.

```ts
it.only("userTotalStaked is broken, user lose funds", async function(){
  //Step2: Round1 - slash Alice's self and community stake of 80000 each
  await this.identityStaking
  .connect(this.owner)
  .slash(
    this.selfStakers.slice(0, 1),
    this.communityStakers.slice(0, 1),
    this.communityStakees.slice(0, 1),
    80,
  );
  //Step2: Round1 - Alice's community/self stake is 20000 after slashing
  expect(
    (
      await this.identityStaking.communityStakes(
        this.communityStakers[0],
        this.communityStakees[0],
      )
    ).amount,
  ).to.equal(20000);
  //Step2: Round1 - total slashed amount 80000 x 2
  expect(await this.identityStaking.totalSlashed(1)).to.equal(160000);
  //Step3: Round1 - Alice appealed and full slash amount is released 80000 x 2
  await this.identityStaking
  .connect(this.owner)
  .release(this.selfStakers[0], this.selfStakers[0], 80000, 1);

  await this.identityStaking
  .connect(this.owner)
  .release(this.communityStakers[0], this.communityStakees[0], 80000, 1);


  //Step3: Round1 - After release, Alice has full staked balance 100000 x 2 
  expect((await this.identityStaking.selfStakes(this.selfStakers[0])).amount).to.equal(100000);
  expect((await this.identityStaking.communityStakes(this.communityStakers[0],this.communityStakees[0])).amount).to.equal(100000);
  expect(await this.identityStaking.totalSlashed(1)).to.equal(0);

  // Alice's lock expired
  await time.increase(twelveWeeksInSeconds + 1);
  //Step4: Alice trying to withdraw 100000 x 2 from selfStake and communityStake. Tx reverted with underflow error. 
  await  expect((this.identityStaking.connect(this.userAccounts[0]).withdrawSelfStake(100000))).to.be.revertedWithPanic(PANIC_CODES.ARITHMETIC_UNDER_OR_OVERFLOW);
  await  expect((this.identityStaking.connect(this.userAccounts[0]).withdrawCommunityStake(this.communityStakees[0],100000))).to.be.revertedWithPanic(PANIC_CODES.ARITHMETIC_UNDER_OR_OVERFLOW);
  //Step4: Alice could only withdraw 20000 x 2. Alice lost 80000 x 2.
  await this.identityStaking.connect(this.userAccounts[0]).withdrawSelfStake(20000);
  await this.identityStaking.connect(this.userAccounts[0]).withdrawCommunityStake(this.communityStakees[0],20000);


 })
```

### Tools Used

Hardhat

### Recommended Mitigation Steps

In `release()`, also update `userTotalStaked`.

**[nutrina (Gitcoin) confirmed](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/9#issuecomment-1998055081)**

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/9#issuecomment-1999415371):**
 > The warden has shown how, due to incorrect accounting, `userTotalStaked` may end up being less than intended, causing a loss of funds to users.
> 
> Due to the impact, I agree with High Severity.

**[Gitcoin mitigated](https://github.com/code-423n4/2024-03-gitcoin-mitigation?tab=readme-ov-file#mitigations-to-be-reviewed):**
> This [PR](https://github.com/gitcoinco/id-staking-v2/pull/8) fixes the `userTotalStaked` invariant (accounting error) [here](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/9).

**Status:** Mitigation confirmed. Full details in reports from [oakcobalt](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/2), [Stormy](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/9) and [0xDING99YA](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/5).

***

# Low Risk and Non-Critical Issues

For this audit, 2 reports were submitted by wardens detailing low risk and non-critical issues. The [report highlighted below](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/12) by **oakcobalt** received the top score from the judge.

*The following wardens also submitted reports: [EV\_om](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/8).*

## [01] Locks can be extended to an old `unlockTime`

**Instance 1**

In IdentityStaking.sol, self-stake or community-stake lock duration can be extended. However, checks are insufficient and allow a new lock to have the same `unlockTime` as the old lock.

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

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L273

**Instance 2**

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

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L371

### Recommendation

Change to `unlockTime <= selfStakes[msg.sender].unlockTime` to ensure new locktime will be different.

## [02] Redundancy in code comments

In IdentityStaking.sol, there are 2 cases of code comment redundancy. 

In `communityStake()` and `extendCommunityStake()`, change `12-104 weeks and 104 weeks` into `12-104 weeks`.

**Instance 1**

```solidity
|>  /// @dev The duration must be between 12-104 weeks and 104 weeks, and after any existing lock for this staker+stakee
  function communityStake(address stakee, uint88 amount, uint64 duration) external whenNotPaused {
...
```

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L310

**Instance 2**

```solidity
|>  /// @dev The duration must be between 12-104 weeks and 104 weeks, and after any existing lock for this staker+stakee
  ///      The unlock time is calculated as `block.timestamp + duration`
  function extendCommunityStake(address stakee, uint64 duration) external whenNotPaused {
...
```

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L332


### Recommendation

Change `12-104 weeks and 104 weeks` into `12-104 weeks`.

## [03] Dust staking for self or community is allowed, which might cause slash to be more expensive to maintain over time 

A user can stake dust amount for themself or other community members. If the user commits offenses, the intended behavior is `if I have staked GTC on other people, and I have misbehaved, then all my stakes on others will be slashed`. Also, if the stakee commits offenses, then `those particular stakes will be slashed`.

Both scenarios mean that the `staker -> stakee` amounts will be slashed regardless of whether the staker or the stakee commits offenses. When staker takes dusts amount on multiple stakees, or a stakee has many dust amount staked by other, this will make the required `slash()` flows more expansive due to dust value updates.

**Instance 1**

```solidity
  function selfStake(uint88 amount, uint64 duration) external whenNotPaused {
...
    if (amount == 0) {
      revert AmountMustBeGreaterThanZero();
    }
...}
```

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L230-L231

**Instance 2**

```solidity
  function communityStake(address stakee, uint88 amount, uint64 duration) external whenNotPaused {
...
    if (amount == 0) {
      revert AmountMustBeGreaterThanZero();
    }
...
```

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L321-L322

### Recommendation

Prevent dust amount staking.

## [04] `slash()` is vulnerable to `lockAndBurn()` racing, accounting of consecutive slashes might be inconsistent

In normal circumstances, consecutive slash amounts will be rolled over. For example, Alice is slashed in round 1. If Alice is slashed again in round 2, the slash amount from round 1 will be rolled over in round 2. See [this doc](https://github.com/code-423n4/2024-03-gitcoin/blob/main/id-staking-v2/README.md#appendix-b-slashing-in-consecutive-rounds).

However, the above behavior cannot be guaranteed due to a possible permissionless `lockAndBurn()` racing. 
When `slash()` on round 2 is settled before `lockAndBurn()`. The doc-described behavior is preserved:

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

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L448

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

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L455

As seen above, `slash()` submitted close to the end of a round(90 days `burnRoundMinimumDuration`), might be counted either at the next round or current round due to potential `lockAndBurn` racing.  When `slash()` settles at the next round, the user's previous round slash amount will not be rolled over, and the slash amount will be directly burned instead of being preserved for the next round.

### Recommendation

Consider having `lockAndBurn()` access controlled by the protocol to ensure `slash()` and `lockAndBurn()` settle in the correct order.

## [05] Racing conditions between `release()` and `slash()` might cause `release()` tx reverts

Suppose a user is approved for a refund of fines from a previous round. But before `release()` is settled, the user is slashed again in the current round. `release()` and `slash()` racing might occur and result in uncertain behavior:

1. Either `release()` succeeds, the user is first refunded with previous slashed amount and then slashed with a new fine.
2. Or `release()` will revert, user's previous and current slashed amount are combined. The user doesn't receive a refund.

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

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L558

`release()` tx might revert due to `slash()` racing. 

### Recommendation

In this case, `RELEASER_ROLE` and `SLASHER_ROLE` need to be coordinated to prevent random reverts. 

## [06] Distinguish event emitting between slashing self-stakes and slashing community-stakes

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

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L497

### Recommendation

Consider adding `stakee` indexed field in the event of community-stake slashing.

## [07] A user might be slashed with `0` amount fine due to rounding

When a user stake dust amount, `slash()` might round the total `slashedAmount` to `0`, resulting in `0` amount slashing.

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

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L443

### Recommendation

Disallow dust amount staking. 

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/12#issuecomment-1993774280):**
 > [01] - Low<br>
> [02] - Non-Critical<br>
> [03] -  Low<br>
> [04] - Low<br>
> [05] - Low<br>
> [06] - Refactor<br>
> [07] - Low<br>
 > Total: 5 Low, 1 Refactor and 1 Non-Critical<br>
 >
 > Most unique report, fairly won.

## [[08] `release()` is vulnerable to racing conditions against permissionless `lockAndBurn()`, users might lose released refunds](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/7)

*Note: The sponsor team decided to mitigate this QA finding that was outside of the winning QA report. This downgraded issue from the same warden has been included in this report for completeness.* 

### Proof of Concept

A slashing round can only be increased in permissionless `lockAndBurn()`. This subject's protocol controlled `release()` flow to racing conditions against `lockAndBurn()`.

When a user is slashed they have a minimal 90-day appeal period (including following round). Suppose Alice is slashed in round 1 and suppose she appealed in round 2. If her appeal succeeds. `release()` can be called by `RELEASER_ROLE` anywhere in round 2.

The issue arises when the appeal approval is late in round 2 and `release()` is submitted close to the end of round 2 (`lastBurnTimestamp + 90` days). `release()` tx has to be submitted with round 1 as input `slashRound` (`slashRound  == currentSlashRound - 1`). 

```solidity
//id-staking-v2/contracts/IdentityStaking.sol
  function release(
    address staker,
    address stakee,
    uint88 amountToRelease,
    uint16 slashRound
  ) external onlyRole(RELEASER_ROLE) whenNotPaused {
      //@audit note: this condition might cause revert when release() tx settles after `currentSlashRound` is incremented in lockAndBurn()
|>    if (slashRound < currentSlashRound - 1) {
      revert RoundAlreadyBurned();
    }
...
...
```

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L541-L542

Now, suppose Bob submitted `lockAndBurn()` tx, which settles before `release()` tx at the end of round 2(`lastBurnTimestamp` `+` 90 days). `lockAndBurn()` will burn all the slashed amount in round 1 (`currentSlashRound - 1`), which effectively burns Alice's slashed amount. Then, it increments the round to 3. 

```solidity
  function lockAndBurn() external whenNotPaused {
    if (block.timestamp - lastBurnTimestamp < burnRoundMinimumDuration) {
      revert MinimumBurnRoundDurationNotMet();
    }
    uint16 roundToBurn = currentSlashRound - 1;
      //@audit This will first burn total slashed amount in `currentSlashRound - 1`, in the above example, including Alice's slashed amount. Then increment to next round.
|>    uint88 amountToBurn = totalSlashed[roundToBurn];
      
|>    ++currentSlashRound;
    lastBurnTimestamp = block.timestamp;
    if (amountToBurn > 0) {
      if (!token.transfer(burnAddress, amountToBurn)) {
        revert FailedTransfer();
      }
    }
    emit Burn(roundToBurn, amountToBurn);
  }
```

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L513-L515

Then, when `release()` settles, `if (slashRound < currentSlashRound - 1)` will be true, causing tx to revert. Alice lost the refund. 

### Recommended Mitigation Steps

1. Consider making `lockAndBurn()` access-controlled by the protocol, which eliminates uncertainties in the order of `release()` settlements.
2. OR introduce a short delay before `lockAndBurn()` is allowed to ensure the appeal and release flow will have time to settle first. 

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/7#issuecomment-2001898823):**
> Low/Non-Critical seems most appropriate.

*Note: For full discussion, see [here](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/7).*

**[Gitcoin mitigated](https://github.com/code-423n4/2024-03-gitcoin-mitigation?tab=readme-ov-file#mitigations-to-be-reviewed):**
> This [PR](https://github.com/gitcoinco/id-staking-v2/pull/9) fixes the issue.

**Status:** Mitigation confirmed. Full details in reports from [oakcobalt](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/4), [Stormy](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/19) and [0xDING99YA](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/7).

## [[09] 90 day minimum appeal period can be violated by `lockAndBurn` in the edge case of protocol pausing](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/15)

*Note: The sponsor team decided to mitigate this QA finding that was outside of the winning QA report. This downgraded issue from the same warden has been included in this report for completeness.* 

In the edge of the contract pausing and unpausing, the 90-day `minimal appeal period` can be violated. Users might have less than 90 days to finish appeals and lose their refunds.

### Proof of Concept

Based on [readme](https://github.com/code-423n4/2024-03-gitcoin?tab=readme-ov-file#overview), the contract needs to enforce:
> at least a minimum appeal time (`burnRoundMinimumDuration`), during which time slashed users can appeal the slashing decision. Upon successful appeal funds can be restored using the release function of the `IdentityStaking` protocol.

The minimum appeal time / `burnRoundMinimumDuration` is implemented in `lockAndBurn()` by checking `block.timestamp - lastBurnTimestamp` `>=` `burnRoundMinimumDuration`. 

However, if the contract has been paused during the current round, `block.timestamp - lastBurnTimestamp` will also include the duration of the pause, when no `release()` can be issued and the appeal process cannot be finalized. 

Suppose the contract was paused for 2-hours, during which `release()` is locked. When the contract is unpaused, `block.timestamp` crosses the `burnRoundMinimumDuration`. Due to `lockAndBurn()` being permissionless, anyone might call `lockAndBurn()` which effectively burns all slashed amount from `currentSlashRound-1` and starts a new round. Any pending released refunds from `currentSlashRound-1` will be burned.

```solidity
  function lockAndBurn() external whenNotPaused {
    if (block.timestamp - lastBurnTimestamp < burnRoundMinimumDuration) {
      revert MinimumBurnRoundDurationNotMet();
    }
...
```

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L509-L510

In the case above, if the user is slashed at the end of a previous round (`currentSlashRound - 1`), they will have (`90 day - 2 hour`) appeal time to receive refunds. The longer the contract is on pause, the less appeal time. 

### Recommended Mitigation Steps

1. In `pause()` and `unpause()` , record timestamp to store duration of current pause(`pauseDuration`).
2. In `lockAndBurn()`, factor in `pauseDuration` (if non-zero) when checking `burnRoundMinimumDuration`.

### Assessed type

Timing

**[Alex the Entreprenerd (judge) commented](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/15#issuecomment-2001901008):**
> The finding is valid, but the impact seems low. The Admin is the cause of the issue, and they will have to "fix it" themselves, through Coordination at the Social Layer. I believe given our extensive discussions on Admin Privilege, the finding is best qualified as low severity.

*Note: For full discussion, see [here](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/15).*

**[Gitcoin mitigated](https://github.com/code-423n4/2024-03-gitcoin-mitigation?tab=readme-ov-file#mitigations-to-be-reviewed):**
> This [PR](https://github.com/gitcoinco/id-staking-v2/pull/9) fixes the issue.

 **Status:** Mitigation confirmed. Full details in reports from [oakcobalt](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/3), [Stormy](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/18) and [0xDING99YA](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/6).

***

# Gas Optimizations

For this audit, 1 report was submitted by wardens detailing gas optimizations. The [report highlighted below](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/13) by **oakcobalt** received the top score from the judge.

## [G-01] Wasteful operation in the staking flow when `unlockTime` doesn't change

**Total Gas Saved (10000)**

In `selfStake()` and `communityStake()`, a user can stake an additional amount with a future `unlockTime`. 

However, when `unlockTime` doesn't change from the old lock, wasteful state operation is performed. The same unlock time is re-written to `selfStakes` or `communityStakes`.

**Instance 1**

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

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L247

**Instance 2**

```solidity
  function communityStake(address stakee, uint88 amount, uint64 duration) external whenNotPaused {
  ...
      //@audit Gas: wasteful operation when unlockTime == existing lock unlockTime
    communityStakes[msg.sender][stakee].amount += amount;
|>    communityStakes[msg.sender][stakee].unlockTime = unlockTime;
```

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L338

Note: Editing the existing value of a storage slot costs 5000 gas.

### Recommendation

Check `unlockTime` and only write to storage when it differs from the old `unlockTime`.

## [G-02] Emit outside of for-loop to save gas (Not included in bot report)

In `slash()`, `Slash` events are emitted inside for-loops. Each event emitting has an overhead of 375 gas. Consider emitting the event outside of for-loops.

**Instance 1**

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

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L467

**Instance 2**

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

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L497

### Recommendation

Emit events outside of for-loops.

## [G-03] When `slashedAmount` rounds to `0`, wasteful operation in `slash()`

In `slash()`, It's possible that `slashedAmount` might round down to `0`. In this case, wasteful zero value math and storage update will be performed.

Note: Each storage slot edit costs 5000 gas. 

**Instance 1**

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

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L462-L465

**Instance 2**

```solidity
...
      comStake.slashedAmount += slashedAmount;
      comStake.amount -= slashedAmount;

      userTotalStaked[staker] -= slashedAmount;
...
```

https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L492-L495

### Recommendation

Check `slashedAmount` and only update storage when `slahsedAmount` is non-zero.

***

# Audit Analysis

For this audit, 1 analysis report was submitted by wardens. An analysis report examines the codebase as a whole, providing observations and advice on such topics as architecture, mechanism, or approach. The [report highlighted below](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/14) by **oakcobalt** received the top score from the judge.

## Summary

The IdentityStaking protocol enables users to stake GTC tokens for a specific period, serving as collateral for reputation. Stakes can be withdrawn or re-locked after expiration. Stamped tokens in Gitcoin Passport correlate with reputation, impacting a user's humanity score. Misbehaving users risk having their stakes slashed, determined off-chain but executed through the protocol. Slashing rules include slashing any staked GTC, regardless of lock period, and slashing stakes placed on oneself or others for misconduct. Slashed GTC is held for an appeal period before potential restoration or burning.

### Existing Standards

- UUPS pattern and EIP1967 standard.
- ERC165 standard.

Current contracts in scope is intended to be compatible with off-chain process of determining slashing eligibility.

## Approach

- Scope: `id-staking-v2/contracts/IdentityStaking.sol` and `id-staking-v2/contracts/IIdentityStaking.sol`.
- Roles: Role-specific flows are focused including `RELEASER_ROLE`, `SLAHSER_ROLE`, `PAUSER_ROLE` and `DEFAULT_ADMIN_ROLE`. Any potential DOS or storage conflicts that might caused by various access-controlled flows are analyzed.
- Upgrade: The upgrade process is reviewed.
- Staking/Slash/Release Scenarios: Various cases of staking/slashing/release flow combination are considered to explore edge cases and possible racing conditions.

## Centralization Risks

Here's an analysis of potential centralization risks in the provided contracts:

### IdentityStaking.sol

- `DEFAULT_ADMIN_ROLE`: This role is by default the admin role for all other roles including self.
- Slashing criteria: slashing eligibility check is not implemented on-chain and `SLAHSER_ROLE` has the centralized ability to slash any user of any percentage of fines.
- Release criteria: The current releasing eligibility check is not implemented on-chain and `RELEASER_ROLE` has the centralized ability to refund or not refund any amount of previous fines.
- Pausing mechanism: `PAUSER_ROLE` can pause all main contract functions including withdrawal. A regular user might not be able to withdraw funds even their balance is unlocked if the contract is paused.
- Upgrade: Current implementation can be upgraded by `DEFAULT_ADMIN_ROLE` at any time.

## Systemic Risks

### Single point of failure of compromised protocol accounts

Multiple protocol trusted role controlled process means if one of the protocol controlled accounts is compromised, the key protocol slashing or releasing flow can be hijacked. With lack of on-chain check on the eligibility of slashing or releasing criteria, there are multiple ways of abusing the system on-chain with a compromised `SLAHSER_ROLE` or `RELEASER_ROLE` account.

Consider adding on-chain checks of slashing and releasing eligibility.

### Various cases of transaction racing

The current implementation allows slash rounds to be incremented by a permissionless `lockAndBurn()` function, which opens up risk of `lockAndBurn()` racing against protocol-controlled flow including `slash()` and `release()`.

Due to no clear incentives for calling permissionless `lockAndBurn()` , there are risks of `lockAndBurn()` being called at uncertain intervals. 

Various pending `slash()` and `release()` transactions of the current slash round might settle with unexpected behavior if `lockAndBurn()` settles first and increments the slash round. 

The `slash()` and `release()` flow might also be subject to unexpected racing against each other. The behavior of whether a user's consecutive fines should be combined or separated might not be consistent.

## Mechanism Review

### Slashing of pending unlocked or unlocked funds

Any funds locked or unlocked are subject to be slashed if a staker commits offenses. However, current `slash()` implementation doesn’t ensure this will be executed as intended. When slashing is determined for a user with pending unlocked or unlocked funds, slashing tx is at risk of user front-run to escape penalty. A 2-step withdrawal process can be considered to mitigate such risks.

## Conclusion

The IdentityStaking protocol offers users the ability to stake GTC tokens, utilizing them as collateral for reputation. This mechanism, integrated with Gitcoin Passport, plays a significant role in determining users' humanity scores. However, the protocol faces various risks, particularly concerning centralization and systemic vulnerabilities. The absence of on-chain checks for slashing and releasing eligibility poses a single point of failure, potentially compromising protocol accounts.

Moreover, the lack of clear incentives for certain functions like **`lockAndBurn()`** introduces the risk of being called at uncertain intervals, potentially leading to unintended consequences such as transaction racing. While the protocol's ability to slash or release funds is critical for maintaining integrity, its current implementation requires enhancements to ensure reliable execution and mitigate risks of abuse or exploitation. Implementing on-chain checks for slashing and releasing eligibility and considering a 2-step withdrawal process could strengthen the protocol's robustness and resilience against potential threats

### Time spent

24 hours

**[nutrina (Gitcoin) commented](https://github.com/code-423n4/2024-03-gitcoin-findings/issues/14#issuecomment-2005885105):**
 > Just adding some information on how plan to address some risks:
> 
> Regarding the risk of centralisation, our plan is to mitigate this by only assigning roles to safe accounts, that are controlled by multiple addresses, and have a minimum number of required signers for any transaction that will be executed.
>
> Regarding `lockAndBurn()` (this is also covered in another issue), this will be mitigated by making the `lockAndBurn()` function require a `BURNER` role, in order to mitigate racing executions against `slash()` or `release()` (releasers, slashers and burners will need to coordinate).

***

# [Mitigation Review](#mitigation-review)

## Introduction

Following the C4 audit, 3 wardens ([oakcobalt](https://code4rena.com/@oakcobalt), [0xDING99YA](https://code4rena.com/@0xDING99YA) and [Stormy](https://code4rena.com/@Stormy)) reviewed the mitigations for all identified issues. Additional details can be found within the [C4 Gitcoin Passport Mitigation Review repository](https://github.com/code-423n4/2024-03-gitcoin-mitigation).

## Overview of Changes

**[Summary from the Sponsor](https://github.com/code-423n4/2024-03-gitcoin-mitigation?tab=readme-ov-file#overview-of-changes):**

The changes include:

1. Fix for the high priority issue found (H-01).
2. Fixes for some of the QA level issues.
3. new code:
    - small changes related to events.
    - convenience functions that we have added to be able to manage multiple community stakes in 1 transaction (create, extend and withdraw multiple stakes).

## Mitigation Review Scope

| URL      | Mitigation of | Purpose      |
| ----------- | ------------- | ------------------- |
| https://github.com/gitcoinco/id-staking-v2/pull/8 | H-01 | This fixes the `userTotalStaked` invariant (accounting error) https://github.com/code-423n4/2024-03-gitcoin-findings/issues/9 |
| https://github.com/gitcoinco/id-staking-v2/pull/9 | QA-09, QA-08 | This fixes the following: https://github.com/code-423n4/2024-03-gitcoin-findings/issues/15, https://github.com/code-423n4/2024-03-gitcoin-findings/issues/7 |
| https://github.com/gitcoinco/id-staking-v2/pull/12 | N/A (new code) | This adds a missing `Release` event and changes the `Slash` event. |
| https://github.com/gitcoinco/id-staking-v2/pull/10 | N/A (new code)  | This adds convenience functions to handle multiple community stakes in 1 call: `multipleCommunityStakes`, `extendMultipleCommunityStake` and `withdrawMultipleCommunityStake`. |

## Mitigation Review Summary

### Individual PRs

| Original Issue | Status | Full Details |
| --- | --- | --- |
| H-01 |  🟢 Mitigation Confirmed | Reports from [oakcobalt](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/2), [Stormy](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/9) and [0xDING99YA](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/5) |
| QA-09 (previous Issue #15) |  🟢 Mitigation Confirmed | Reports from [oakcobalt](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/3), [Stormy](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/18) and [0xDING99YA](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/6) |
| QA-08 (previous Issue #7) |  🟢 Mitigation Confirmed | Reports from [oakcobalt](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/4), [Stormy](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/19) and [0xDING99YA](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/7) |
| New Code: PR-10 |  🟢 Mitigation Confirmed | Reports from [oakcobalt](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/12), [Stormy](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/16) and [0xDING99YA](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/14) |
| New Code: PR-12 |  🟢 Mitigation Confirmed | Reports from [oakcobalt](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/11) and [0xDING99YA](https://github.com/code-423n4/2024-03-gitcoin-mitigation-findings/issues/13) |

**During the mitigation review, the wardens confirmed that all in-scope findings were mitigated.**

***

# Disclosures

C4 is an open organization governed by participants in the community.

C4 audits incentivize the discovery of exploits, vulnerabilities, and bugs in smart contracts. Security researchers are rewarded at an increasing rate for finding higher-risk issues. Audit submissions are judged by a knowledgeable security researcher and solidity developer and disclosed to sponsoring developers. C4 does not conduct formal verification regarding the provided code but instead provides final verification.

C4 does not provide any guarantee or warranty regarding the security of this project. All smart contract software should be used at the sole risk and responsibility of users.

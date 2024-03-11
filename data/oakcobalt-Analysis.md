## Summary

The IdentityStaking protocol enables users to stake GTC tokens for a specific period, serving as collateral for reputation. Stakes can be withdrawn or re-locked after expiration. Stamped tokens in Gitcoin Passport correlate with reputation, impacting a user's humanity score. Misbehaving users risk having their stakes slashed, determined off-chain but executed through the protocol. Slashing rules include slashing any staked GTC, regardless of lock period, and slashing stakes placed on oneself or others for misconduct. Slashed GTC is held for an appeal period before potential restoration or burning.

**Existing Standards:**

- UUPS pattern and  EIP1967 standard
- ERC165 standard

Current contracts in scope is intended to be compatible with off-chain process of determining slashing eligibility.

## 1- Approach:

- Scope: id-staking-v2/contracts/IdentityStaking.sol, id-staking-v2/contracts/IIdentityStaking.sol
- Roles: Role-specific flows are focused including `RELEASER_ROLE`, `SLAHSER_ROLE`, `PAUSER_ROLE` and `DEFAULT_ADMIN_ROLE`. Any potential DOS or storage conflicts that might caused by various access-controlled flows are analyzed.
- Upgrade: The upgrade process is reviewed.
- Staking/Slash/Release Scenarios: Various cases of staking/slashing/release flow combination are considered to explore edge cases and possible racing conditions.

## 2- Centralization Risks:

Here's an analysis of potential centralization risks in the provided contracts:

### IdentityStaking.sol

- `DEFAULT_ADMIN_ROLE`: This role is by default the admin role for all other roles including self.
- Slashing criteria: slashing eligibility check is not implemented on-chain and `SLAHSER_ROLE` has the centralized ability to slash any user of any percentage of fines.
- Release criteria: The current releasing eligibility check is not implemented on-chain and `RELEASER_ROLE` has the centralized ability to refund or not refund any amount of previous fines.
- Pausing mechanism: `PAUSER_ROLE` can pause all main contract functions including withdraw. A regular user might not be able to withdraw funds even their balance is unlocked if the contract is paused.
- Upgrade: Current implementation can be upgraded by `DEFAULT_ADMIN_ROLE` at any time.

## 3- Systemic Risks:

### Single point of failure of compromised protocol accounts

Multiple protocol trusted role controlled process means if one of the protocol controlled accounts is compromised, the key protocol slashing or releasing flow can be hijacked. With lack of on-chain check on the eligibility of slashing or releasing criteria, there are multiple ways of abusing the system on-chain with a compromised `SLAHSER_ROLE` or `RELEASER_ROLE` account.

Consider adding on-chain checks of slashing and releasing eligibility.

### Various cases of transaction racing

The current implementation allows slash rounds to be incremented by a permissionless `lockAndBurn()` function, which opens up risk of `lockAndBurn()` racing against protocol-controlled flow including `slash()`, `release()`

(1) Due to no clear incentives for calling permissionless `lockAndBurn()` , there are risks of `lockAndBurn()` being called at uncertain intervals. 

Various pending `slash()` and `release()` transactions of the current slash round might settle with unexpected behavior if `lockAndBurn()` settles first and increments the slash round. 

(2) `slash()` and `release()` flow might also be subject to unexpected racing against each other. The behavior of whether a user's consecutive fines should be combined or separated might not be consistent.

## 4- Mechanism Review:

### Slashing of pending unlocked or unlocked funds

Any funds locked or unlocked are subject to be slashed if a staker commits offenses. However, current `slash()` implementation doesn’t ensure this will be executed as intended. When slashing is determined for a user with pending unlocked or unlocked funds, slashing tx is at risk of user front-run to escape penalty. A 2-step withdrawal process can be considered to mitigate such risks.

## Conclusion:

The IdentityStaking protocol offers users the ability to stake GTC tokens, utilizing them as collateral for reputation. This mechanism, integrated with Gitcoin Passport, plays a significant role in determining users' humanity scores. However, the protocol faces various risks, particularly concerning centralization and systemic vulnerabilities. The absence of on-chain checks for slashing and releasing eligibility poses a single point of failure, potentially compromising protocol accounts. Moreover, the lack of clear incentives for certain functions like **`lockAndBurn()`** introduces the risk of being called at uncertain intervals, potentially leading to unintended consequences such as transaction racing. While the protocol's ability to slash or release funds is critical for maintaining integrity, its current implementation requires enhancements to ensure reliable execution and mitigate risks of abuse or exploitation. Implementing on-chain checks for slashing and releasing eligibility and considering a 2-step withdrawal process could strengthen the protocol's robustness and resilience against potential threats

### Time spent:
24 hours
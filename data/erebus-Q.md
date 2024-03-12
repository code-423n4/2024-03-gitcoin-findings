## Low

### [L-01] Missing `_disableInitializers()` call in the constructor of the implementation

It is best practice to prevent a malicious user from initializing the implementation of a proxy, so a call in the constructor to `_disableInitializers()` should be made:

[IdentityStaking.sol#L172](https://github.com/code-423n4/2024-03-gitcoin/blob/6529b351cd72a858541f60c52f0e5ad0fb6f1b16/id-staking-v2/contracts/IdentityStaking.sol#L172)

```diff
+    constructor() {
+        _disableInitializers();
+    }
```
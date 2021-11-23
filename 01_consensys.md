# Consensys Diligence Audit Findings
___
## Aave Protocol V2 Audit

[Report](https://consensys.net/diligence/audits/2020/09/aave-protocol-v2)

### 1. Aave Protocol V2

**Finding**: Unhandled return values of `transfer` and `transferFrom`

**Description**: ERC20 implementations are not always consistent. Some implementations of `transfer` and `transferFrom` could return `false` on failure instead of reverting. It is safer to wrap such calls into `require` statements to catch these failures.

**Recommendation**: Check the return value and revert on `0` or `false`, or use OpenZeppelin’s `SafeERC20` wrapper functions

**Severity**: <span style="color:black; background-color:yellow">Medium</span>
___
## Defi Saver Audit

[Report](https://consensys.net/diligence/audits/2021/03/defi-saver)

### 2. Defi Saver

**Finding**: Random task execution

**Description**: In a scenario where a user takes a flash loan, `_parseFLAndExecute()` gives the flash loan wrapper contract (`FLAaveV2`, `FLDyDx`) the permission to execute functions on behalf of the user’s `DSProxy`. This execution permission is revoked only after the entire recipe execution is finished, which means that in case that any of the external calls along the recipe execution is malicious, it might call `executeAction()` back, i.e. Reentrancy Attack, and inject any task it wishes (e.g. take user’s funds out, drain approved tokens, etc)

**Recommendation**: A reentrancy guard (mutex) should be used to prevent such attack

**Severity**: <span style="color:black; background-color:red">Critical</span>

### 3. Defi Saver (2)

**Finding**: Tokens with more than 18 decimal points will cause issues

**Description**: It is assumed that the maximum number of decimals for each token is 18. However uncommon, it is possible to have tokens with more than 18 decimals, as an example `YAMv2` has 24 decimals. This can result in broken code flow and unpredictable outcomes

**Recommendation**: Make sure the code won’t fail in case the token’s decimals is more than 18

**Severity**: <span style="color:black; background-color:orange">Major</span>


### 4. Defi Saver (3)

**Finding**: Error codes of Compound’s `Comptroller.enterMarket` and `Comptroller.exitMarket` are not checked

**Description**: Compound’s `enterMarket` and `exitMarket` functions return an error code instead of reverting in case of failure. DeFi Saver smart contracts never check for the error codes returned from Compound smart contracts.

**Recommendation**: Caller contract should `revert` for non-zero error code

**Severity**: <span style="color:black; background-color:orange">Major</span>
___
## DAOfi Audit

[Report](https://consensys.net/diligence/audits/2021/03/defi-saver)

### 5. DAOfi

**Finding**: Reversed order of parameters in allowance function call

**Description**: the parameters that are used for the allowance function call are not in the same order that is used later in the call to `safeTransferFrom`.

**Recommendation**: Reverse the order of parameters in allowance function call to fit the order that is in the `safeTransferFrom` function call.

**Severity**: <span style="color:black; background-color:yellow">Medium</span>

### 6. DAOfi (2)

**Finding**: Token approvals can be stolen in `DAOfiV1Router01.addLiquidity`

**Description**: `DAOfiV1Router01.addLiquidity` creates the desired pair contract if it does not already exist, then transfers tokens into the pair and calls `DAOfiV1Pair.deposit`. There is no validation of the address to transfer tokens from, so an attacker could pass in any address with nonzero token approvals to `DAOfiV1Router`. This could be used to add liquidity to a pair contract for which the attacker is the `pairOwner`, allowing the stolen funds to be retrieved using `DAOfiV1Pair.withdraw`.

**Recommendation**: Transfer tokens from `msg.sender` instead of `lp.sender`

**Severity**: <span style="color:black; background-color:red">Critical</span>

### 7. DAOfi (3)

**Finding**: `swapExactTokensForETH` checks the wrong return value

**Description**: Instead of checking that the amount of tokens received from a swap is greater than the minimum amount expected from this swap, it calculates the difference between the initial receiver’s balance and the balance of the router

**Recommendation**: Check the intended values

**Severity**: <span style="color:black; background-color:orange">Major</span>

### 8. DAOfi (4)

**Finding**: `DAOfiV1Pair.deposit` accepts deposits of zero, blocking the pool

**Description**: `DAOfiV1Pair.deposit` is used to deposit liquidity into the pool. Only a single deposit can be made, so no liquidity can ever be added to a pool where `deposited == true`. The `deposit` function does not check for a nonzero deposit amount in either token, so a malicious user that does not hold any of the `baseToken` or `quoteToken` can lock the pool by calling `deposit` without first transferring any funds to the pool.

**Recommendation**: Require a minimum deposit amount with non-zero checks

**Severity**: <span style="color:black; background-color:yellow">Medium</span>
___
## Fei Protocol Audit

[Report](https://consensys.net/diligence/audits/2021/01/fei-protocol)

### 9. Fei Protocol

**Finding**: `GenesisGroup.commit` overwrites previously committed values

**Description**: The amount stored in the recipient’s `committedFGEN` balance overwrites any previously committed value. Additionally, this also allows anyone to commit an amount of `0` to any account, deleting their commitment entirely.

**Recommendation**: Ensure the committed amount is added to the existing commitment.

**Severity**: <span style="color:black; background-color:red">Critical</span>

### 10. Fei Protocol (2)

**Finding**: Purchasing and committing still possible after launch

**Description**: Even after `GenesisGroup.launch` has successfully been executed, it is still possible to invoke `GenesisGroup.purchase` and `GenesisGroup.commit`.

**Recommendation**: Consider adding validation in `GenesisGroup.purchase` and `GenesisGroup.commit` to make sure that these functions cannot be called after the launch.

**Severity**: <span style="color:black; background-color:red">Critical</span>

### 11. Fei Protocol (3)

**Finding**: `UniswapIncentive` overflow on pre-transfer hooks

**Description**: Before a token transfer is performed, Fei performs some combination of mint/burn operations via `UniswapIncentive.incentivize`. Both `incentivizeBuy` and `incentivizeSell` calculate incentives using overflow-prone math, then mint or burn from the target according to the results. This may have unintended consequences, like allowing a caller to mint tokens before transferring them, or burn tokens from their recipient.

**Recommendation**: Ensure casts in `getBuyIncentive` and `getSellPenalty` do not overflow

**Severity**: <span style="color:black; background-color:orange">Major</span>

### 12. Fei Protocol (4)

**Finding**: `BondingCurve` allows users to acquire FEI before launch

**Description**: `allocate` can be called before genesis launch, as long as the contract holds some nonzero PCV. By force-sending the contract 1 wei, anyone can bypass the majority of checks and actions in `allocate`, and mint themselves FEI each time the timer expires.

**Recommendation**: Prevent `allocate` from being called before genesis launch

**Severity**: <span style="color:black; background-color:yellow">Medium</span>

### 13. Fei Protocol (5)

**Finding**: `Timed.isTimeEnded` returns `true` if the timer has not been initialized

**Description**: Timed initialization is a 2-step process:
1. `Timed.duration` is set in the `constructor`
2. `Timed.startTime` is set when the method `_initTimed` is called. Before this second method is called, `isTimeEnded()` calculates remaining time using a `startTime` of `0`, resulting in the method returning `true` for most values, even though the timer has not technically been started.

**Recommendation**: If `Timed` has not been initialized, `isTimeEnded()` should return `false`, or `revert`

**Severity**: <span style="color:black; background-color:yellow">Medium</span>

### 14. Fei Protocol (6)

**Finding**: Overflow and underflow protection

**Description**: Having overflow and underflow vulnerabilities is very common for smart contracts. It is usually mitigated by using `SafeMath` or using solidity `version ^0.8`. In this code, many arithmetical operations are used without the ‘safe’ version. The reasoning behind it is that all the values are derived from the actual ETH values, so they can’t overflow.

**Recommendation**: In our opinion, it is still safer to have these operations in a safe mode. So we recommend using `SafeMath` or solidity `version ^0.8` compiler.

**Severity**: <span style="color:black; background-color:yellow">Medium</span>

### 15. Fei Protocol (7)

**Finding**: Unchecked return value for `IWETH.transfer` call

**Description**: In `EthUniswapPCVController`, there is a call to `IWETH.transfer` that does not check the return value. It is usually good to add a `require` statement that checks the return value or to use something like `safeTransfer`, unless one is sure the given token reverts in case of a failure.

**Recommendation**: Consider adding a `require` statement, or using `safeTransfer`

**Severity**: <span style="color:black; background-color:yellow">Medium</span>

### 16. Fei Protocol (8)

**Finding**: `GenesisGroup.emergencyExit` remains functional after launch

**Description**: `emergencyExit` is intended as an escape mechanism for users in the event the genesis `launch` method fails or is frozen. `emergencyExit` becomes callable 3 days after launch is callable. These two methods are intended to be mutually-exclusive, but are not: either method remains callable after a successful call to the other. This may result in accounting edge cases.

**Recommendation**:
1. Ensure `launch` cannot be called if `emergencyExit` has been called
2. Ensure `emergencyExit` cannot be called if `launch` has been called

**Severity**: <span style="color:black; background-color:yellow">Medium</span>
___
## bitbank Audit

[Report](https://consensys.net/diligence/audits/2020/11/bitbank)

### 17. bitbank

**Finding**: ERC20 tokens with no return value will fail to transfer

**Description**: Although the ERC20 standard suggests that a `transfer` should return `true` on success, many tokens are non-compliant in this regard. In that case, the `transfer` call here will revert even if the transfer is successful, because solidity will check that the `RETURNDATASIZE` matches the ERC20 interface.

**Recommendation**: Consider using OpenZeppelin’s `SafeERC20`

**Severity**: <span style="color:black; background-color:orange">Major</span>
___
## MetaSwap Audit

[Report](https://consensys.net/diligence/audits/2020/08/metaswap)

### 18. MetaSwap

**Finding**: Reentrancy vulnerability in `MetaSwap.swap`

**Description**: If an attacker is able to reenter `swap`, they can execute their own trade using the same tokens and get all the tokens for themselves.

**Recommendation**: Use a simple reentrancy guard, such as OpenZeppelin’s `ReentrancyGuard` to prevent reentrancy in `MetaSwap.swap`

**Severity**: <span style="color:black; background-color:orange">Major</span>

### 19. MetaSwap

**Finding**: A new malicious adapter can access users’ tokens

**Description**: The purpose of the `MetaSwap` contract is to save users gas costs when dealing with a number of different aggregators. They can just `approve` their tokens to be spent by `MetaSwap` (or in a later architecture, the `Spender` contract). They can then perform trades with all supported aggregators without having to reapprove anything. A downside to this design is that a malicious (or buggy) adapter has access to a large collection of valuable assets. Even a user who has diligently checked all existing adapter code before interacting with `MetaSwap` runs the risk of having their funds intercepted by a new malicious adapter that’s added later.

**Recommendation**: Make `MetaSwap` contract the only contract that receives token approval. It then moves tokens to the `Spender` contract before that contract `DELEGATECALLs` to the appropriate adapter. In this model, newly added adapters shouldn’t be able to access users’ funds.

**Severity**: <span style="color:black; background-color:yellow">Medium</span>

### 20. MetaSwap (2)

**Finding**: Owner can front-run traders by updating adapters

**Description**: `MetaSwap` owners can front-run users to swap an adapter implementation. This could be used by a malicious or compromised owner to steal from users. Because adapters are `DELEGATECALL`’ed, they can modify `storage`. This means any adapter can overwrite the logic of another adapter, regardless of what policies are put in place at the contract level. Users must fully trust every adapter because just one malicious adapter could change the logic of all other adapters.

**Recommendation**: At a minimum, disallow modification of existing adapters. Instead, simply add new adapters and disable the old ones.

**Severity**: <span style="color:black; background-color:yellow">Medium</span>
___
## mstable-1.1 Audit

[Report](https://consensys.net/diligence/audits/2020/07/mstable-1.1)

### 21. mstable-1.1

**Finding**: Users can collect interest from `SavingsContract` by only staking `mTokens` momentarily

**Description**: The `SAVE` contract allows users to deposit `mAssets` in return for lending yield and swap fees. When depositing `mAsset`, users receive a “credit” tokens at the momentary credit/mAsset exchange rate which is updated at every deposit. However, the smart contract enforces a minimum timeframe of 30 minutes in which the interest rate will not be updated. A user who deposits shortly before the end of the timeframe will receive credits at the stale interest rate and can immediately trigger an update of the rate and withdraw at the updated (more favorable) rate after the 30 minutes window. As a result, it would be possible for users to benefit from interest payouts by only staking `mAssets` momentarily and using them for other purposes the rest of the time.

**Recommendation**: Remove the 30 minutes window such that every deposit also updates the exchange rate between credits and tokens.

**Severity**: <span style="color:black; background-color:yellow">Medium</span>
___
## Bancor v2 AMM Audit

[Report](https://consensys.net/diligence/audits/2020/06/bancor-v2-amm-security-audit)

### 22. Bancor v2 AMM

**Finding**: Oracle updates can be manipulated to perform atomic front-running attack

**Description**: It is possible to atomically arbitrage rate changes in a risk-free way by “sandwiching” the `Oracle` update between two transactions. The attacker would send the following 2 transactions at the moment the `Oracle` update appears in the mempool:
1. The first transaction, which is sent with a higher gas price than the `Oracle` update transaction, converts a very small amount. This “locks in” the conversion weights for the block since `handleExternalRateChange` only updates weights once per block. By doing this, the arbitrageur ensures that the stale Oracle price is initially used when doing the first conversion in the following transaction.
2. The second transaction, which is sent at a slightly lower gas price than the transaction that updates the `Oracle`, performs a large conversion at the old weight, adds a small amount of Liquidity to trigger rebalancing and converts back at the new rate. The attacker can obtain liquidity for step 2 using a flash loan. The attack will deplete the reserves of the pool.

**Recommendation**: Do not allow users to trade at a stale `Oracle` rate and trigger an `Oracle` price update in the same transaction.

**Severity**: <span style="color:black; background-color:red">Critical</span>
___
## Shell Protocol Audit

[Report](https://consensys.net/diligence/audits/2020/06/shell-protocol)

### 23. Shell Protocol

**Finding**: Certain functions lack input validation routines

**Description**: The functions should first check if the passed arguments are valid first. These checks should include, but not be limited to:
1. `uint` should be larger than `0` when `0` is considered invalid
2. `uint` should be within constraints
3. `int` should be positive in some cases
4. length of arrays should match if more arrays are sent as arguments
5. addresses should not be `0x0`

**Recommendation**: Add tests that check if all of the arguments have been validated. Consider checking arguments as an important part of writing code and developing the system.

**Severity**: <span style="color:black; background-color:orange">Major</span>

### 24. Shell Protocol (2)

**Finding**: Remove `Loihi` methods that can be used as backdoors by the administrator

**Description**: There are several functions in `Loihi` that give extreme powers to the shell administrator. The most dangerous set of those is the ones granting the capability to add assimilators. Since assimilators are essentially a proxy architecture to delegate code to several different implementations of the same interface, the administrator could, intentionally or unintentionally, deploy malicious or faulty code in the implementation of an assimilator. This means that the administrator is essentially totally trusted to not run code that, for example, drains the whole pool or locks up the users’ and LPs’ tokens. In addition to these, the function `safeApprove` allows the administrator to move any of the tokens the contract holds to any address regardless of the balances any of the users have. This can also be used by the owner as a backdoor to completely drain the contract.

**Recommendation**:
1. Remove the `safeApprove` function and, instead, use a trustless escape hatch mechanism.
2. For the assimilator addition functions, our recommendation is that they are made completely internal, only callable in the `constructor`, at deploy time. Even though this is not a big structural change (in fact, it reduces the attack surface), it is, indeed, a feature loss. However, this is the only way to make each shell a time-invariant system. This would not only increase Shell’s security but also would greatly improve the trust the users have in the protocol since, after deployment, the code is now static and auditable.

**Severity**: <span style="color:black; background-color:orange">Major</span>
___
## Lien Protocol Audit

[Report](https://consensys.net/diligence/audits/2020/05/lien-protocol)

### 25. Lien Protocol

**Finding**: A reverting `fallback` function will lock up all payouts

**Description**: In `BoxExchange.sol`, the internal function `_transferEth` reverts if the transfer does not succeed. The `_payment` function processes a list of transfers to settle the transactions in an `ExchangeBox`. If any of the recipients of an ETH transfer is a smart contract that reverts, then the entire payout will fail and will be unrecoverable.

**Recommendation**:
1. Implement a queuing mechanism to allow buyers and sellers to initiate the withdrawal on their own using a *pull-over-push* pattern.
2. Ignore a failed transfer and leave the responsibility up to users to receive them properly.

**Severity**: <span style="color:black; background-color:red">Critical</span>
___
## The Lao Audit

[Report](https://consensys.net/diligence/audits/2020/01/the-lao)

### 26. The Lao

**Finding**: `Saferagequit` makes you lose funds

**Description**: `safeRagequit` and `ragequit` functions are used for withdrawing funds from the LAO. The difference between them is that `ragequit` function tries to withdraw all the allowed tokens and `safeRagequit` function withdraws only some subset of these tokens, defined by the user. It’s needed in case the user or `GuildBank` is blacklisted in some of the tokens and the transfer reverts. The problem is that even though you can quit in that case, you’ll lose the tokens that you exclude from the list. To be precise, the tokens are not completely lost, they will belong to the LAO and can still potentially be transferred to the user who quit. But that requires a lot of trust, coordination, and time, and anyone can steal some part of these tokens.

**Recommendation**: Implementing *pull pattern* for token withdrawals should solve the issue. Users will be able to quit the LAO and burn their shares but still keep their tokens in the LAO’s contract for some time if they can’t withdraw them right now.

**Severity**: <span style="color:black; background-color:red">Critical</span>

### 27. The Lao (2)

**Finding**: Creating proposal is not trustless

**Description**: Usually, if someone submits a proposal and transfers some amount of tribute tokens, these tokens are transferred back if the proposal is rejected. But if the proposal is not processed before the emergency processing, these tokens will not be transferred back to the proposer. This might happen if a tribute token or a deposit token transfers are blocked. Tokens are not completely lost in that case, they now belong to the LAO shareholders and they might try to return that money back. But that requires a lot of coordination and time and everyone who ragequits during that time will take a part of that tokens with them.

**Recommendation**: *Pull pattern* for token transfers would solve the issue

**Severity**: <span style="color:black; background-color:red">Critical</span>


### 28. The Lao (3)

**Finding**: Emergency processing can be blocked

**Description**: The main reason for the emergency processing mechanism is that there is a chance that some token transfers might be blocked. For example, a sender or a receiver is in the USDC blacklist. Emergency processing saves from this problem by not transferring tribute token back to the user (if there is some) and rejecting the proposal. The problem is that there is still a deposit transfer back to the sponsor and it could be potentially blocked too. If that happens, proposal can’t be processed and the LAO is blocked.

**Recommendation**: *Pull pattern* for token transfers would solve the issue

**Severity**: <span style="color:black; background-color:red">Critical</span>


### 29. The Lao (4)

**Finding**: Token Overflow might result in system halt or loss of funds

**Description**: If a token overflows, some functionality such as `processProposal`or `cancelProposal` will break due to `SafeMath` reverts. The overflow could happen because the supply of the token was artificially inflated to oblivion.

**Recommendation**: We recommend to allow overflow for broken or malicious tokens. This is to prevent system halt or loss of funds. It should be noted that in case an overflow occurs, the balance of the token will be incorrect for all token holders in the system

**Severity**: <span style="color:black; background-color:orange">Major</span>

### 30. The Lao (5)

**Finding**: Whitelisted tokens limit

**Description**: `_ragequit` function is iterating over all whitelisted tokens. If the number of tokens is too big, a transaction can run out of gas and all funds will be blocked forever.

**Recommendation**: A simple solution would be just limiting the number of whitelisted tokens. If the intention is to invest in many new tokens over time, and it’s not an option to limit the number of whitelisted tokens, it’s possible to add a function that removes tokens from the whitelist. For example, it’s possible to add a new type of proposal that is used to vote on token removal if the balance of this token is zero. Before voting for that, shareholders should sell all the balance of that token.

**Severity**: <span style="color:black; background-color:orange">Major</span>

### 31. The Lao (6)

**Finding**: Summoner can steal funds using `bailout`

**Description**: The `bailout` function allows anyone to transfer kicked user’s funds to the summoner if the user does not call `safeRagequit` (which forces the user to lose some funds). The intention is for the summoner to transfer these funds to the kicked member afterwards. The issue here is that it requires a lot of trust to the summoner on the one hand, and requires more time to kick the member out of the LAO.

**Recommendation**: By implementing *pull pattern* for token transfers, kicked member won’t be able to block the `ragekick` and the LAO members would be able to kick anyone much quicker. There is no need to keep the `bailout` function.

**Severity**: <span style="color:black; background-color:orange">Major</span>

### 32. The Lao (7)

**Finding**: Sponsorship front-running

**Description**: If proposal submission and sponsorship are done in 2 different transactions, it’s possible to front-run the `sponsorProposal` function by any member. The incentive to do that is to be able to block the proposal afterwards.

**Recommendation**: *Pull pattern* for token transfers will solve the issue. Front-running will still be possible but it doesn’t affect anything.

**Severity**: <span style="color:black; background-color:orange">Major</span>

### 33. The Lao (8)

**Finding**: Delegate assignment front-running

**Description**: Any member can front-run another member’s `delegateKey` assignment. If you try to submit an address as your `delegateKey`, someone else can try to assign your delegate address to themselves. While incentive of this action is unclear, it’s possible to block some address from being a delegate forever.

**Recommendation**: Make it possible for a `delegateKey` to approve `delegateKey` assignment or cancel the current delegation. Commit-reveal methods can also be used to mitigate this attack.

**Severity**: <span style="color:black; background-color:yellow">Medium</span>

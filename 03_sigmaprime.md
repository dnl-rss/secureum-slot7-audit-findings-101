# Sigma Prime Audit Findings
___
## Ether Collateral Audit

[Report](https://github.com/sigp/public-audits/blob/master/synthetix/ethercollateral/review.pdf)

### 67. Ether Collateral

<span style="color:black; background-color:orange">High</span>

**Finding**: Improper Supply Cap Limitation Enforcement

**Description**: The `openLoan()` function does not check if the loan to be issued will result in the supply cap being exceeded. It only enforces that the supply cap is not reached before the loan is opened. As a result, any account can create a loan that exceeds the maximum amount of sETH that can be issued by the `EtherCollateral` contract.

**Recommendation**: Introduce a `require` statement in the `openLoan()` function to prevent the total cap from being exceeded by the loan to be opened.



### 68.Ether Collateral

**Finding**: Improper Storage Management of Open Loan Accounts

**Description**: When loans are open, the associated account address gets added to the `accountsWithOpenLoans` array regardless of whether the account already has a loan/is already included in the array. Additionally, it is possible for a malicious actor to create a denial of service condition exploiting the unbound storage array in `accountsSynthLoans`.

**Recommendation**:
1. Consider changing the `storeLoan` function to only push the account to the `accountsWithOpenLoans` array if the loan to be stored is the first one for that particular account ;
2. Introduce a limit to the number of loans each account can have.

<span style="color:black; background-color:orange">High</span>


### 69. Ether Collateral

<span style="color:black; background-color:yellow">Medium</span>

**Finding**: Contract Owner Can Arbitrarily Change Minting Fees and Interest Rates

**Description**: The `issueFeeRate` and `interestRate` variables can both be changed by the `EtherCollateral` contract owner after loans have been opened. As a result, the owner can control fees such as they equal/exceed the collateral for any given loan.

**Recommendation**: While "dynamic" interest rates are common, we recommend considering the minting fee ( `issueFeeRate` ) to be a constant that cannot be changed by the owner.


___
## InfiniGold Audit

[Report](https://github.com/sigp/public-audits/raw/master/infinigold/review.pdf)

### 70. InfiniGold

<span style="color:black; background-color:red">Critical</span>

**Finding**: Inadequate Proxy Implementation Preventing Contract Upgrades:

**Description**: The `TokenImpl` smart contract requires `Owner` , `name` , `symbol` and `decimals` of `TokenImpl` to be set by the `TokenImpl` `constructor`. Consider two smart contracts, contract A and contract B . If contract A performs a `delegatecall` on contract B , the state/storage variables of contract B are not accessible by contract A . Therefore, when `TokenProxy` targets an implementation of `TokenImpl` and interacts with it via a `DELEGATECALL` , it will not be able to access any of the state variables of the `TokenImpl` contract. Instead, the `TokenProxy` will access its local storage, which does not contain the variables set in the constructor of the `TokenImpl` implementation. When the `TokenProxy` contract is constructed it will only initialize and set two storage slots:
- The proxy admin address ( _setAdmin internal function)
- The token implementation address ( _setImplementation private function)
Hence when a proxy call to the implementation is made, variables such as `Owner` will be uninitialised (effectively set to their default value). This is equivalent to the owner being the 0x0 address. Without access to the implementation state variables, the proxy contract is rendered unusable.

**Recommendation**:
1. Set fixed constant parameters as Solidity constants. The solidity compiler replaces all occurrences of a constant in the code and thus does not reserve state for them. Thus if the correct getters exist for the ERC20 interface, the proxy contract doesn’t need to initialise anything.
2. Create a constructor-like function that can only be called once within `TokenImpl`. This can be used to set the state variables as is currently done in the constructor, however if called by the proxy after deployment, the proxy will set its state variables.
3. Create `getter` and `setter` functions that can only be called by the `owner`. Note that this strategy allows the owner to change various parameters of the contract after deployment.
4. Predetermine the slots used by the required variables and set them in the `constructor` of the proxy. The storage slots used by a contract are deterministic and can be computed. Hence the variables `Owner` , `name` , `symbol` and `decimals` can be set directly by their slot in the proxy constructor.


### 71. InfiniGold

<span style="color:black; background-color:orange">High</span>

**Finding**: Blacklisting Bypass via `transferFrom()` Function

**Description**: The `transferFrom()` function in the `TokenImpl` contract does not verify that the sender (i.e. the `from` address) is not blacklisted. As such, it is possible for a user to allow an account to spend a certain allowance regardless of their blacklisting status.

**Recommendation**: At present the function `transferFrom()` uses the `notBlacklisted(address)` modifier twice, on the `msg.sender` and `to` addresses. The `notBlacklisted(address)` modifier should be used a third time against the `from` address.


___
## Synthetix Unipool Audit

[Report](https://github.com/sigp/public-audits/blob/master/synthetix/unipool/review.pdf)

### 72. Synthetix Unipool

<span style="color:black; background-color:red">Critical</span>

**Finding**: Wrong Order of Operations Leads to Exponentiation of `rewardPerTokenStored`

**Description**: `rewardPerTokenStored` is mistakenly used in the numerator of a fraction instead of being added to the fraction. The result is that `rewardPerTokenStored` will grow exponentially thereby severely overstating each individual's rewards earned. Individuals will therefore either be able to withdraw more funds than should be allocated to them or they will not be able to withdraw their funds at all as the contract has insufficient SNX balance. This vulnerability makes the Unipool contract unusable.

**Recommendation**: Adjust the function `rewardPerToken()` to represent the original functionality.


### 73. Synthetix Unipool

<span style="color:black; background-color:orange">High</span>

**Finding**: Staking Before Initial `notifyRewardAmount` Can Lead to Disproportionate Rewards

**Description**: If a user successfully stakes an amount of UNI tokens before the function `notifyRewardAmount` is called for the first time, their initial `userRewardPerTokenPaid` will be set to zero. The staker would be paid out funds greater than their share of the SNX rewards.

**Recommendation**: We recommend preventing `stake` from being called before `notifyRewardAmount` is called for the first time.



### 74. Synthetix Unipool

<span style="color:black; background-color:orange">High</span>

**Finding**: External Call Reverts if Period Has Not Elapsed

**Description**: The function `notifyRewardAmount` will revert if `block.timestamp >= periodFinish`. However this function is called indirectly via the `Synthetix.mint` function. A revert here would cause the external call to fail and thereby halt the mint process. `Synthetix.mint` cannot be successfully called until enough time has elapsed for the period to finish.

**Recommendation**: Consider handling the case where the reward period has not elapsed without reverting the call.



### 75. Synthetix Unipool

<span style="color:black; background-color:yellow">Medium</span>

**Finding**: Gap Between Periods Can Lead to Erroneous Rewards

**Description**: The SNX rewards are earned each period based on reward and duration as specified in the `notifyRewardAmount` function. The contract will output more rewards than it receives. Therefore if all stakers call `getReward` the contract will not have enough SNX balance to transfer out all the rewards and some stakers may not receive any rewards.

**Recommendation**: We recommend enforcing each period start exactly at the end of the previous period.


___
## Chainlink Audit

[Report](https://github.com/sigp/public-audits/blob/master/chainlink-1/review.pdf)

### 76. Chainlink

<span style="color:black; background-color:orange">High</span>

**Finding**: Malicious Users Can DOS/Hijack Requests From Chainlinked Contracts

**Description**: Malicious users can hijack or perform Denial of Service (DOS) attacks on requests of Chainlinked contracts by replicating or front-running legitimate requests. The Chainlinked (`Chainlinked.sol`) contract contains the `checkChainlinkFulfillment()` modifier. This modifier is demonstrated in the examples that come with the repository. In these examples this modifier is used within the functions which contracts implement that will be called by the Oracle when fulfilling requests. It requires that the caller of the function be the Oracle that corresponds to the request that is being fulfilled. Thus, requests from `Chainlinked` contracts are expected to only be fulfilled by the Oracle that they have requested. However, because a request can specify an arbitrary callback address, a malicious user can also place a request where the callback address is a target `Chainlinked` contract. If this malicious request gets fulfilled first (which can ask for incorrect or malicious results), the Oracle will call the legitimate contract and fulfil it with incorrect or malicious results. Because the known requests of a `Chainlinked` contract gets deleted, the legitimate request will fail. It could be such that the Oracle fulfils requests in the order in which they are received. In such cases, the malicious user could simply front-run the requests to be higher in the queue.

**Recommendation**: This issue arises due to the fact that any request can specify its own arbitrary callback address. A restrictive solution would be where callback addresses are localised to the requester themselves.

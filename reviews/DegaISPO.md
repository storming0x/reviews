# Code Review Report for DEGA ISPO Contracts

# Introduction

A time-boxed security review of the **DEGA** ISPO smart contracts was done by **storm0x**, with a focus on the security and risk aspects of the implementation.

# Disclaimer

A smart contract security review can never verify the complete absence of vulnerabilities. This is a time, resource and expertise bound effort where I try to find as many vulnerabilities as possible. I can not guarantee 100% security after the review or even if the review will find any problems with your smart contracts. Subsequent security reviews, bug bounty programs and on-chain monitoring are strongly recommended.

# About **storm0x**

**storm0x**, is an independent smart contract security researcher, core contributor at yearnfi and SEAL-911 contributor. He mostly contributes at yearn and the ethereum security community giving talks, developing FOSS tools and tweeting about smart contract development and security best practices. Reach out on Twitter [@storming0x](https://twitter.com/storming0x)
[farcaster](https://warpcast.com/storming0x)

# About **DEGA ISPO Contracts**

Staking contracts for ISPO in ethereum for Liquid Staking ETH and Matic tokens using stETH rebasing token and stMatic. The staking contracts separate the yield from the underlying tokens from users that deposit into the contracts, the users receive their original underlying staked amount on withdrawal and accumulate a reward token earned based on staking amounts. The reward tokens claiming are calculated off-chain outside smart contracts and is not in scope for this review.

**Additional References used during review** (Link to underlying protocols or relevant documents):

- [Lido stEth](https://github.com/lidofinance/lido-dao/blob/master/contracts/0.4.24/StETH.sol)
- [Lido stMatic](https://github.com/lidofinance/polygon-contracts/blob/main/contracts/StMATIC.sol)
- [Lido 1-2 wei corner](https://docs.lido.fi/guides/steth-integration-guide/#1-2-wei-corner-case)
- [Lido deployed contracts](https://docs.lido.fi/deployed-contracts/)

# Threat Model

For DEGA ISPO contracts:

- Unprivileged user can call the `deposit` `withdraw`, `emergencyWithdraw` (if enabled), `assignRewards` state changing functions to interact with the contracts.``
- Privileged user can only directly call `pause`, `unpause`, `setMaxTotalDeposit`, `enableEmergencyWithdraw` and `adminWithdraw`.
- The contract's external calls interact with the Lido contracts [stETH ethereum address (proxy)](https://etherscan.io/address/0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84) and [stMATIC (proxy)](https://etherscan.io/address/0x9ee91F9f426fA633d227f7a9b000E28b9dfd8599) calling `getPooledEthByShares`, `getSharesByPooledEth`, `sharesOf`, `transferShares`, `transferSharesFrom`, `transfer`, `convertMaticToStMatic`, `convertStMaticToMatic`, `balanceOf`

## Privileged Roles & Actors

- Pause Role - Can pause and unpause the contracts disabling withdrawals and deposits.
- Default Admin Role - Can withdraw the treasury shares calculated by the contract, enable emergency withdraw and change the deposit max limit.

## Observations

The testing suite helped a lot during this code review, i think the contracts can use some more testing in fork mode with the real contracts targeted in production around reward calculations, and multiple user deposits, the mocks have some assumptions that need to be validated in production. Another approach is testing live in production before launch to be sure integration is correct.

Most of the code assumes that the Lido Contracts share value can only increase, but there's a theoretical scenario that given a big slashing event the Lido Contracts could lose share value which these contracts need to handle for possible edge case.

# Severity classification

| Severity               | Impact: High | Impact: Medium | Impact: Low |
| ---------------------- | ------------ | -------------- | ----------- |
| **Likelihood: High**   | Critical     | High           | Medium      |
| **Likelihood: Medium** | High         | Medium         | Low         |
| **Likelihood: Low**    | Medium       | Low            | Low         |

**Impact** - the technical, economic and reputation damage of a successful attack

**Likelihood** - the chance that a particular vulnerability gets discovered and exploited

**Severity** - the overall criticality of the risk

# Security Assessment Summary

**_review commit hash_ - [7df73dd](https://github.com/DEGAorg/DEGA-ISPO/commit/7df73dd)**

**DEGA team post review change commit hash\_ - [dd24eb6](https://github.com/DEGAorg/DEGA-ISPO/commit/dd24eb6b922eb055f89614b80bc6cc8e22e708c9)**

### Scope

The following smart contracts were in scope of the audit:

- `DegaISPO.sol`
- `DegaISPOPolygon.sol`

The following number of issues were found, categorized by their severity:

- Critical & High: 3 issues
- Medium: 4 issues
- Low: 1 issues
- Info: 2 issues

---

# Findings Summary

| ID     | Title                                                                                           | Severity      |
| ------ | ----------------------------------------------------------------------------------------------- | ------------- |
| [C-01] | Admin can drain all shares from contract                                                        | Critical      |
| [H-02] | emergencyWithdraw doesn't account for potential losses on staking contracts                     | High          |
| [H-03] | Operational risk of locked funds in case Pause Role is hijacked                                 | High          |
| [M-01] | Incorrect return value check after token transfers                                              | Medium        |
| [M-02] | First depositor edge cases getting more rewards and 0 debt                                      | Medium        |
| [M-03] | Missing tolerance check on conversion rate when depositing                                      | Medium        |
| [M-04] | Rewards calculation does not handle correctly a loss in share price                             | Medium        |
| [L-01] | getStakedBalance may not show the correct amount staked by user in case of share price decrease | Low           |
| [I-01] | Solidity Style Guidelines can be implemented                                                    | Informational |
| [I-02] | Various comments and optimizations                                                              | Informational |

Rewards calculation does not handle correctly a loss in share price

# Detailed Findings

# [C-01] Admin can drain all shares from contract

## Severity

**Impact:** High

**Likelihood:** Medium

## Description

In both contracts for ISPO, DEGA admin withdraw allows admin to withdraw entire amount of shares from the contract without restriction.

https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/Ethereum/DegaISPO.sol#L82

https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/polygon/DegaISPOPolygon.sol#L83

state variable `degaTreasuryShares` tracks the amount of shares the treasury can take from the contract and with the check:

`require(sharesToWithdraw <= degaTreasuryShares, "Insufficient shares to withdraw amount from treasury");`

It checks that the shares withdrawn cant be higher than that amount, but theres no update to the `degaTreasuryShares` variable after a successful admin withdrawal, hence the trx can be repeated over and over until all shares of the contract have been drained.

This also poses a risk to users and admins of the contract in case of admin keys being stolen.

## Recommendations

Update the state of the withdrawn degaTreasuryShares when adminWithdraw is called.

# [H-01] emergencyWithdraw doesn't account for potential losses on staking contracts

## Severity

**Impact:** High

**Likelihood:** Low

## Description

The function `emergencyWithdraw` in both `DegaISPO.sol` and `DegaISPOPolygon.so`l doesn't socialize "theoretical" losses correctly. In the case emergencyWithdraw is enabled and a loss of ether happens in the Lido protocol, the users that first exit the DegaISPO contract will not incurr any loss from their initial deposit amount, and the last users to withdraw will eat the losses of the first withdrawers.

Explanation:

The contract uses as price of oracle for shares the lido contract calls `getSharesByPooledEth` and `getPooledEthByShares` on normal deposit and withdrawal operations.

Given that Lido states here https://docs.lido.fi/guides/steth-integration-guide/#accounting-oracle
on rebasing in case of a slashing event in theory the totalPooledEth could be lower affecting these numbers.

If two users deposited 1 ETH with the following values stored at the time of deposit for each user would be:
user.shares = 887508165087906362
user.amount = 1000000000000000000

In the scenario where the pooledEther in lido has been lowered to e.g 10000 ETH (loss of slashing or hack) from when the user deposited and emergencyWithdraw is enabled.

The first user can call emergencyWithdraw which will use the `user.amount = 1000000000000000000`
And the shares needed to withdraw this amount would be higher than `user.shares` since there is less eth in the pool lido contract.

The emergency withdraw would incorrectly send the user the complete requested amount and debit shares from the total shares affecting the users remaining.

This issue is not possible for the normal `withdraw` function since there's an additional check that quoted shares are not more than what the `user.shares` value is.

`require(user.shares >= sharesToWithdraw, "Insufficient shares to withdraw amount");`

# [H-02] Operational risk of locked funds in case Pause Role is hijacked

## Severity

**Impact:** High

**Likelihood:** Low

## Description

Users of both ISPO contracts have to trust that the PAUSER_ROLE does not lock the funds permanently or admins of the contract don't lose their keys in a pause state which funds may never be recovered from, not even with `emergencyWithdraw`. Operational risk and management of those admin keys need to be highly secure or via a high threshold multi-sig to avoid potentially locking of funds.

## Recommendations

A design change for this unlikely, but high risk scenario would be good to take into consideration or evaluate threat model for it.

# [M-01] Incorrect return value check after token transfers

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

In `DegalIspo.sol` for the methods `emergencyWithdraw` and `withdraw` it checks the return values to be >=0

https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/Ethereum/DegaISPO.sol#L174

https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/Ethereum/DegaISPO.sol#L193

This condition will never be false, also an important check is missing to compare the amount return from transfers to amountToWithdraw from users.

NOTE: this issue assumes the `getSharesByPooledEth` calls are correct.

## Recommendations

Check what are the values for a failed transfer and edit condition.

# [M-02] First depositor edge cases getting more rewards and 0 debt.

## Severity

**Impact:** Medium

**Likelihood:** High

## Description

First depositor has special edge cases given that `assignrewards` method would have 0 value in `totalStakeTokenDeposited`.

https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/Ethereum/DegaISPO.sol#L135

This means that both the user.debt would be 0 for first depositor. Later when a second depositor deposits and triggers the `assignRewards` , the first depositor will get more rewards than other users since user.debt = 0.

https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/polygon/DegaISPOPolygon.sol#L211

and the debt is not substracted from rewards from him as long as they never withdraw after first deposit. If this first deposit is a big amount of the staking contracts they could get considerably more rewards than others.

## Recommendations

Add tests with multiple user depositing and calculating rewards with similar amounts to check if there's an advantage for first depositor in calculations to validate the code is correct in this edge case.

# [M-03] Missing tolerance check on conversion rate when depositing

## Severity

**Impact:** Medium

**Likelihood:** Low

## Description

On the function `deposit` for both contract there's a missing a tolerance check on conversion of shares to depositedAmount.

A TOLERANCE factor of 1-2 wei can be added to deposit method, as stated by lido docs here https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/Ethereum/DegaISPO.sol#L137

As an exercise i checked the contract https://etherscan.io/address/0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84#readProxyContract

and simulated a deposit of 10ETH at the time of this test

```
uint256 depositStakeShares = lidoContract.getSharesByPooledEth(_amount);
uint256 finalDepositedAmount = lidoContract.getPooledEthByShares(depositStakeShares);
```

Where `depositStakeShares = 8875081650879063621` and `finalDepositedAmount = 9999999999999999999`
because of the wei loss it may be useful to check the `finalDepositedAmount` and `_amount` given by user is never bigger in difference than 2 wei as Tolerance. In case there's an issue with Lido contracts in an upgrade or future versions at least the user won't incur in a loss because of bad state on stETH pooled ETH.

Also as good practice the deposit method should return the `finalDepositedAmount` to user.
And add to the natspec this edge case and in line here
https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/Ethereum/DegaISPO.sol#L137
the reference to docs that the user will lose 1-2 wei in deposit.

## Recommendations

Add tolerance check and return the value of the actual deposited amount so that users depositing are clear what's their final staked amount. This is another method that can benefit from live fork testing with the real contracts.

# [M-04] Rewards calculation does not handle correctly a loss in share price

## Severity

**Impact:** Medium

**Likelihood:** Low

## Description

As mentioned in issue H-01, there's a theoretical scenario for losses to happen in both Lido contracts, e.g in the contract `DegaISPO.sol` method `rewardsCalculation` here:

https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/Ethereum/DegaISPO.sol#L246

in case of a loss the amount of `currentStAmount` may be lower than `totalStakeTokenDeposited` after a loss happens in the Lido protocol, but there's a scenario where `accTokenPerShare > 0` because of a previous rewards calculation, the value is never reset to a lower value since here this calculation:

https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/Ethereum/DegaISPO.sol#L228

the `accTokenPerShare` value will never be adjusted since the right side of the formula becomes 0.

## Recommendations

Mock testing scenarios where potential losses in share value happen and check reward amounts adjustments.

# [L-01] getStakedBalance may not show the correct amount staked by user in case of share price decrease

## Severity

**Impact:** Low

**Likelihood:** Low

## Description

In both contracts the `getStakedBalance` view function according to docs

> @dev Retrieves the staked balance of a user.

But this may not be the case if there's a loss scenario in the lido contracts, that stake amount only tracks the deposited amount but may not be longer valid in the unlikely scenario of a loss.

# [I-01] Solidity Style Guidelines can be implemented

## Description

the contracts should follow the order here for solidity style guide if possible.

https://docs.soliditylang.org/en/v0.8.17/style-guide.html#order-of-functions

Here:https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/Ethereum/DegaISPO.sol#L245

Its personal preference and good practice to go with `_rewardsCalculation` to denote internal functions with prefix underscore so by accident an internal function is never made external.

MAX_TOTAL_DEPOSIT is not a constant so style should for state var should be `maxTotalDeposit`

# [I-02] Various comments and optimizations

## Recommendations

- The following variables could be immutable in constructor to save gas on storage size on both ISPO contracts.

```
uint256 public PRECISION_FACTOR;
ILido public lidoContract;
```

- MAX_TOTAL_DEPOSIT can be zero in constructor, missing zero check for value in both contracts.

- DegaIspo.sol although it reverts on both fallback and receive for ether, is still possible to send ether to it via a self destruct contract, this may not be anissue since self destruct is being deprecated soon from the EVM.

- Consider making a the `assignRewards` function permissioned since currently its safe to be called by anyone but in the future changes may be introduced that put state changing variables at risk. Also the share loss extreme scenarios may merit further check if this function should be public.

- For gas optimization can remove unused constant REWARD_SCHEDULER_ROLE since it does not seem to be used in both contracts.

Also here
https://github.com/DEGAorg/DEGA-ISPO/blob/7df73dd7711db0c04c9b7a72aa2062699c4580cf/contracts/Ethereum/DegaISPO.sol#L227

you can check if `calculationResults.stakedTokenRewardAmount = 0 `and avoid the rest of the calculations to save some gas.

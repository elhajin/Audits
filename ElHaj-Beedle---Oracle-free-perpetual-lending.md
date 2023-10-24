# Beedle - Oracle free perpetual lending - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. buyLoan function is broken ](#H-01)
    - ### [H-02. No check for transferFrom() returned value.](#H-02)




# <a id='contest-summary'></a>Contest Summary

### Sponsor: BeedleFi

### Dates: Jul 24th, 2023 - Aug 7th, 2023

[See more contest details here](https://www.codehawks.com/contests/clkbo1fa20009jr08nyyf9wbx)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 0
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. buyLoan function is broken             

### Relevant GitHub Links
	
https://github.com/elhajin/Audit/blob/main/02-buyLoan-bug.md

# Vulnerability Report: Unauthorized Loan Purchase and Collateral Withdrawal

## Overview

This vulnerability report highlights a security flaw identified in the lending system's in function`buyLoan`. The vulnerability allows a malicious actor to exploit the auction system, manipulate pool balances, and gain unauthorized ownership of loans, potentially leading to significant financial losses and disruption of the lending platform.

## Vulnerability Description

The vulnerability arises due to insufficient validation and checks within the `buyLoan` function. It enables the malicious actor to buy a loan from the auction using a `poolId` that they do not own (i.e., they are not the lender of the pool). As a result, the function incorrectly updates the loan's ownership to the malicious actor's address without requiring any collateral transfer.

## Exploitation Steps

1. **Unauthorized Loan Purchase:**

   - The malicious actor identifies a loan in the auction and decides to purchase it.
   - Using the `buyLoan` function, the malicious actor passes a `poolId` that they do not own (they are not the lender of this pool).
     and also may not Have the same loanTokens and collateralTokens ,since there are no checks for mismatching tokens which gives the malicious actor a lot of choice from pools.
   - As a result, the function updates the loan ownership to the malicious actor's address while taking the debt from the pool provided by the malicious actor.

2. **Unintended Pool Balance Update:**

   - Since the malicious actor has used a `poolId` that they do not own, the function incorrectly updates the pool balance of this pool.
   - The pool balance is decreased by the total debt amount (loan debt + lender interest + protocol interest) without proper checks, causing a loss of funds from the pool.And the decrease in balance will never be paid to this pool again.

3. **Loan Auction Manipulation:**

   - Now that the malicious actor owns the loan, they can create a new pool with the same `loanToken` and `collateralToken` as the stolen loan.
   - The malicious actor places the loan from the previous step into the new pool for auction.(he can set the auction as short as he want)
   - However, due to the incorrect pool balance in the new pool, no one will be able to buy this loan from the auction. cause this will always revert :

   ```sh
     pools[oldPoolId].outstandingLoans -= loan.debt.
   ```

4. **Collateral Withdrawal and Loan Repayment:**
   _if you understanding the contract, you may now think_ ,**_but how the malicious actor can withdraw this funds, since his `outstandingLoans` is zero ❓❓_**</br>
   well :
   - After the auction is done, the malicious actor puts the loan token into their pool as collateral.
   - The malicious actor borrows the same amount (or max close) of debt from their pool by providing the minimum amount of callateral(1 token)and he will be able to do that by setting the `maxLoanRatio` to a large amount.
   - With the pool now holding the loanTokens and the malicious actor holding the loan debt, the outstanding loan balance in the pool increases accordingly.
   - The malicious actor then calls the `seizeLoan` function to withdraw the maximum amount of collateral from the stolen loan.
   - No the malicious actor end up with the borrowed tokens (that equal to the collateral that he provide or little less when he borrow), and the collateral stolen.
     > `NOTICE` The malicious actor may do this in one transaction to avoid front running

## Impact

The impact of this vulnerability is severe and multi-faceted:

1. **Financial Losses:** The incorrect pool balance updates lead to a loss of funds from the targeted pool, causing financial losses for legitimate users and liquidity providers.

2. **Unauthorized Loan Ownership:** The malicious actor gains unauthorized ownership of loans without providing the required collateral, undermining the security and integrity of the lending platform.

3. **Auction Manipulation:** The malicious actor can manipulate the auction system, making certain loans unobtainable by others, potentially disrupting the loan market.

4. **Misuse of Loan Tokens:** The malicious actor obtains loan tokens as a borrower without intending to repay them, leading to a reduction in available loan tokens and affecting the lending platform's stability.

## Recommended Mitigation Steps

in the function `buyLoan` check:

```solidity
    if (msg.sender != pools[poolId].lender) rever;
    if (loans[loanId].loanToken != pools[poolId].loanToken) revert;
    if (loans[loanId].collateralToken != pools[poolId].collateralToken) revert;
```

## **POC**

A proof of concept (PoC) test is available in the contract's repository: [testLender.t.sol](https://github.com/elhajin/Audit/blob/main/test/buyLoan.sol)


## <a id='H-02'></a>H-02. No check for transferFrom() returned value.            

### Relevant GitHub Links
	
https://github.com/elhajin/Audit

## Summary
- The vulnerability in the `lender` smart contract arises from the improper handling of tokens with specific ERC20 tokens, 
  such as ZRX,BAT.. which return false instead of throwing when the `transferFrom()` function fails. Additionally, 
  the contract fails to check the return values of the `transferFrom()` function.

- The impact of this vulnerability includes the unauthorized creation of lending pools without providing any tokens as 
  collateral **with any balance**. Malicious actors can then drain all the funds in these pools (represented by the token) without providing any tokens.
  

- To address this vulnerability, the contract should implement proper return value checks or use `safeTransferFrom()` .
## Vulnerability Details

## Impact
This vulnerability poses a significant risk to the security and integrity of the lending pool system. The scenario involves a malicious lender who takes advantage of certain ERC20 tokens that return `false` instead of throwing an exception when the `transferFrom()` function fails. Additionally, the lender contract fails to properly check the return values of the `transferFrom()` function.

The potential impact of this vulnerability is as follows:

1. **Unauthorized Pool Creation:** The malicious lender exploits the vulnerability to create a lending pool without sending any tokens to the pool. As a result, the pool is initialized without the required collateral, which violates the intended functionality of the lending protocol.

2. **Token Drainage:** By bypassing the token transfer during pool creation, the malicious lender can drain all the tokens provided by other legitimate pools. Since the pool was initialized without proper collateral, the attacker can exercise their options and withdraw the tokens offered by other lenders without providing any corresponding tokens in return. This action leads to a loss of funds for the legitimate lenders, resulting in financial damages and destabilization of the lending ecosystem.

3. **Loss of User Funds:** Users who participate in the affected lending pools may suffer substantial financial losses. The attacker's ability to drain tokens from other pools could result in significant losses for lenders and borrowers who expected the lending pools to operate securely and according to the intended rules.

4. **Reputation Damage:** The presence of such a vulnerability can severely damage the reputation and trustworthiness of the lending platform. Users may lose confidence in the platform's security measures, leading to a decreased willingness to use the lending service.
## Tools Used
Manuel review.
## Recommendations
Check the `success` boolean of all transferFrom() calls. Alternatively, use openzeppelin SafeERC20’s `safeTransferFrom()` function.
		






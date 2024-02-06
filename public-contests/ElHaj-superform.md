### Lack of Token Validation in Cross-Chain Deposit Flow

**Severity:** High risk

**Context:** _(No context files were provided by the reviewer)_

- links :

- [superFormRouter](https://github.com/superform-xyz/superform-core/blob/main/src/BaseRouterImplementation.sol)
- [updateDepositPayload](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L83C14-L83C34)
- [processPayload](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L196)
- [\_singleDeposit](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L878)
- [xChainDepositIntoVault](https://github.com/superform-xyz/superform-core/blob/main/src/forms/ERC4626FormImplementation.sol#L246)

- Summary :

The cross-chain deposit flow lacks a crucial validation step to verify that the token deposited by the user matches the underlying asset of the target superForm. This gap allows a user to initiate an `xChainDeposit` with a less valuable token (e.g., dai) while providing a `superFormId` of a superForm with a more valuable underlying asset (e.g., WETH).resulting into this user depositing more value then he sent.

- Vulnerability Details :

The vulnerability unfolds as follows:

1. On the source chain, a user initiates an [singleXChainSingleVaultDeposit()](https://github.com/superform-xyz/superform-core/blob/main/src/BaseRouterImplementation.sol#L128) with any token and provides a `superFormId` of a superForm whose underlying asset is different then the token the user suppliying and more valuable.

2. The [`superRouter`](https://github.com/superform-xyz/superform-core/blob/main/src/BaseRouterImplementation.sol#L128) conducts checks and, assuming no swap is set by the user supplied data (should be no swap in dstChain for this attack to work). The user's token is then farwarded through the bridge given to the `CoreStateRegistry` contract along with the message and its proofs.

3. Upon token arrival, the `CORE_STATE_REGISTRY_UPDATER_ROLE` keeper observes the amount received and updates the user's `payloadId` by calling [updateDepositPayload](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L83C14-L83C34) after checking against the user given slippage .

   > as the sponsore said the keeper track the received amount of tokens to the dstChain and update it once it received in the coreStateRegistry .

4. The `CORE_STATE_REGISTRY_PROCESSOR_ROLE` keeper then came to process the payload by calling [`processPayload`](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L147). Given that the action is a deposit and the callback type is `INIT`, the [\_singleDeposit](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L878) function is executed within the `CoreStateRegistry` contract.

   ```js
   else if (txType == uint8(TransactionType.DEPOSIT)) {
                   if (initialState != PayloadState.UPDATED) {
                       revert Error.PAYLOAD_NOT_UPDATED();
                   }

                   returnMessage = isMulti == 1
                       ? _multiDeposit(payloadId_, payloadBody_, srcSender, srcChainId)
   >>               : _singleDeposit(payloadId_, payloadBody_, srcSender, srcChainId);
               }
   ```

5. The [\_singleDeposit](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L878) function retrieves the superForm address using the user-supplied `superFormId` from the source chain,and obtains the underlying asset via the `getVaultAsset()` function, and proceeds without validating the match between the user-supplied token and the superForm's underlying asset.

```js
  function _singleDeposit(uint256 payloadId_,bytes memory payload_,address srcSender_,uint64 srcChainIdfrom txData_ )internal
        returns (bytes memory){
            // decode the user supplied payload after the updated amount..
        InitSingleVaultData memory singleVaultData = abi.decode(payload_, (InitSingleVaultData));
        (address superform_,,) = singleVaultData.superformId.getSuperform();
    >>  IERC20 underlying = IERC20(IBaseForm(superform_).getVaultAsset());
    >>  if (underlying.balanceOf(address(this)) >= singleVaultData.amount) {
    >>      underlying.safeIncreaseAllowance(superform_, singleVaultData.amount);
    >>      try IBaseForm(superform_).xChainDepositIntoVault(singleVaultData, srcSender_, srcChainId_) returns (
                uint256 dstAmount
            ) {
    >>          if (dstAmount != 0 && !singleVaultData.retain4626) {
                    return _singleReturnData(
                        srcSender_,
                        singleVaultData.payloadId,
                        TransactionType.DEPOSIT,
                        CallbackType.RETURN,
                        singleVaultData.superformId,
                        dstAmount
                    );
                }
            } catch {
                // some code if deposit fail ..
        } else {
            revert Error.BRIDGE_TOKENS_PENDING();
        }
        return "";
    }
  }
```

note that the asset address is Obtained from the `superFormId` , and the contract check it's balance against this asset.also it's increasing the allowance for this asset , and then call [xChainDepositIntoVault](https://github.com/superform-xyz/superform-core/blob/main/src/forms/ERC4626FormImplementation.sol#L246) with the user `singleVaultData`. And till this point there is no check soever if the user supplied token is the same as the underlying asset of the supplied `superFormId`.

- notice that in case of direct deposit the [\_directDepositIntoVault](https://github.com/superform-xyz/superform-core/blob/main/src/forms/ERC4626FormImplementation.sol#L156) function in the superForm contract,Handle this situation properly and check the token that in `singleVaultData` which is the token that the user supplied, against his asset , but this is not the case in crosschain deposit, the `xChainDepositIntoVault` function don't check the user-supplied token against the vault's underlying asset. it trust [coreStateRegistry]() and transfer from the amount that the user set.

```js
 function _processXChainDeposit(
        InitSingleVaultData memory singleVaultData_,
        uint64 srcChainId_
     )
        internal
        returns (uint256 dstAmount)
     {
        (,, uint64 dstChainId) = singleVaultData_.superformId.getSuperform();
        address vaultLoc = vault;
        IERC4626 v = IERC4626(vaultLoc);
    >>  IERC20(asset).safeTransferFrom(msg.sender, address(this), singleVaultData_.amount);
        IERC20(asset).safeIncreaseAllowance(vaultLoc, singleVaultData_.amount);
        if (singleVaultData_.retain4626) {
            dstAmount = v.deposit(singleVaultData_.amount, singleVaultData_.receiverAddress);
        } else {
            dstAmount = v.deposit(singleVaultData_.amount, address(this));
        }
        emit Processed(srcChainId_, dstChainId, singleVaultData_.payloadId, singleVaultData_.amount, vaultLoc);
    }

```

- `NOTICE` The only factor that could prevent this exploit is if the contract does not have the specific valuable asset chosen by the user at the time of the attack.but given that the contract frequently contains a mix of assets due to unprocessed deposits, pending withdrawals, failed deposit ..ect, it is highly probable that the contract has the necessary amount of a more valuable underlying asset at any given time.
- also even in failed deposits the token to be rescued will be the token of the vault which derived from the `supreFormId` given by the user.

- POC

- Scenario :

Assuming an attacker notices that the `CoreStateRegistry` contract on Arbitrum holds a balance of WETH (10), they could execute the following exploit:

1. The attacker initiates an `xChainDeposit` from Polygon to Arbitrum with 10 dai, while falsely specifying a `superFormId` for a superForm whose underlying asset is WETH.
2. the keeper track the transaction and assuming that 9 dai arrived, he update the amount to 9 (which is valid agains user slippage).
3. The deposit is processed without validation, and due to the existing WETH balance on Arbitrum's `CoreStateRegistry`, the contract erroneously uses 9 WETH for the deposit into the vault associated with the provided `superFormId`.
4. As a result, the attacker is credited with shares or value corresponding to 9 WETH (Of course he will set `retainERC4626 = true`), despite having only deposited 10 dai , thus exploiting the system's lack of asset validation.

- i had some problems Compiling the testing setup in the reposo I made a Minimalistic setup Also i ignored the first part when the user sending the token from the srcChain I believe it's pretty clear that there is no validation there, i started from when the `coreStateRegistry` receives the payload. also i mock the `superRegistry` to avoid all Access control errors, and shows in the test that the `coreStateRegistry` are Completely depends on the given superFormId given by the user to derive the token to be deposited is this token and doesn't matter the user supplied token :

```js
 //SPDX-License-Identi MIT
pragma solidity ^0.8.23;
import "forge-std/Test.sol";
import "forge-std/console.sol";
import "../src/SuperPositions.sol";
import {SuperformRouter} from "../src/SuperformRouter.sol";
import "../src/SuperformFactory.sol";
import "../src/forms/ERC4626Form.sol";
import "./mocks/VaultMock.sol";
import "./mocks/MockERC20.sol";
import "../src/crosschain-data/extensions/CoreStateRegistry.sol";
import "../src/interfaces/ISuperRegistry.sol";
contract superRegistryMock {
   // to skip the onlyminter param .
   address spr ;
   address factory;
    function getAddress(bytes32 v ) public view returns(address){
        if (v== keccak256("CORE_STATE_REGISTRY")) return spr;
        else if (v == keccak256("SUPERFORM_FACTORY")) return factory;

        return address(this);
    }
    function hasProtocolAdminRole(address admin_) external view returns (bool){
        return true;
    }
    function getStateRegistry(uint8 a) public view returns (address) {
        if (a == 1) return spr;
        return address(0);
    }
    function isValidAmbImpl(address) public view returns(bool){
        return true;
    }
    function hasRole(bytes32 id_, address addressToCheck_) public view returns(bool){
        return true;
    }
    function setCorestate(address _spr) public {
        spr = _spr;
    }

    function getRequiredMessagingQuorum(uint64) public view returns(uint) {
        return 0;
    }
    function setFactory(address f) public {
        factory =f ;
    }
}
contract setup_poc is Test{
    mapping (address => uint8) stateRegistryIds;
    superRegistryMock sr;
    SuperPositions sp;
    SuperformRouter superRouter;
    SuperformFactory factory;
    ERC4626Form impl;
    VaultMock vault;
    MockERC20 Weth;
    MockERC20 token;
    CoreStateRegistry coreState;
    uint superFormId;
    address superForm;

    function setUp() public {
        // deploy the tokens :
        Weth = new MockERC20("wrapped eth","weth",address(1223),1);
        token = new MockERC20("wrapped eth","weth",address(this),10 ether);
        // deploy the vault : and set the asset as the weth;
        vault = new VaultMock(Weth,"vault1","v1");
        // deploy superRegistry ;
        sr = new superRegistryMock();
        // deploy the implementation :
        impl = new ERC4626Form(address(sr));
        // deploy the factory and add the implementation :
        factory = new SuperformFactory(address(sr));
        sr.setFactory(address(factory));
        // add the implementation :
        factory.addFormImplementation(address(impl),1);
        // create a superform :
        (superFormId,superForm) = factory.createSuperform(1,address(vault));
        // deploy the coreStateRegistry ;
        coreState = new CoreStateRegistry(ISuperRegistry(address(sr)));
        sr.setCorestate(address(coreState));
        // simulate that the coreState have some weth of other users :
        Weth.mint(address(coreState),10 ether);
    }


   function test_callCoreState() public {
        // call the receivePayload sumilate the behavior of amb.(quorm is set to 0 for testing);
        coreState.receivePayload(111,initiateUserInput());
        // the keeper then came and update the amount (assuming arrived 9 token) :
        uint[] memory amounts = new uint[](1);
        amounts[0] = 9 ether;
        coreState.updateDepositPayload(1,amounts);
        // balance of coreState before :
        uint balanceBefore = Weth.balanceOf(address(coreState));
        console.log("coreState before weth balance : " ,balanceBefore);

        // the processor keeper then came and process the payload .
        coreState.processPayload(1);
        console.log("coreState remain weth balance : " ,balanceBefore - Weth.balanceOf(address(coreState)));
        console.log("user minted shares" , vault.balanceOf(address(this)));// address(this) is srcSender;
   }
    // this is the data that core state will receive after the msgDelivered :
    function initiateUserInput() public view returns (bytes memory ambMessage){
        // this is just the data after it get parsed and sent through the amb in xchainSingleDeposit():
        LiqRequest memory liq ;//empty
        bytes memory ambData = abi.encode(InitSingleVaultData(3,superFormId,10 ether,1000,liq,false,true,address(this),""));
        uint8[] memory ambIds = new uint8[](1);
        ambIds[0] = 1;
         ambMessage =  abi.encode(AMBMessage(DataLib.packTxInfo(
                uint8(TransactionType.DEPOSIT),
                uint8(CallbackType.INIT),
                0,
                1,
                address(this), // msg.sender
                111
            ), abi.encode(ambIds,ambData)));
    }
}
```

- Impact

- This vulnerability can be exploited to deposit assets into a vault that do not match the user's provided token, leading to the issuance of shares based on an incorrect asset and value. This can result in significant financial losses to other users.

- Recommendation

- I would recommend that the keeper update the token received from the user also not only the amount , so it can be compared To the targeted vault asset.

## Medium risk

### `retainERC4626` Flag mishandling in `_updateMultiDeposit`

**Severity:** Medium risk

**Context:** _(No context files were provided by the reviewer)_

- Severity: mid

- src issue : [CoreStateRegistry : \_updateMultiDeposit()](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L386)

- Description

- after a multiDeposit cross-chain is arrived to the `coreStateRegistry` , the keeper will update the deposit first , before process it through [updateDepositPayload](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L83) function .

```solidity
  function updateDepositPayload(uint256 payloadId_, uint256[] calldata finalAmounts_) external virtual override {
        _onlyAllowedCaller(keccak256("CORE_STATE_REGISTRY_UPDATER_ROLE"));

        // some code ....

        PayloadState finalState;
        if (isMulti != 0) {
    >>        (prevPayloadBody, finalState) = _updateMultiDeposit(payloadId_, prevPayloadBody, finalAmounts_);
        } else {
            // this will may or may not update the amount n prevPayloadBody .
            (prevPayloadBody, finalState) = _updateSingleDeposit(payloadId_, prevPayloadBody, finalAmounts_[0]);
        }
        // some code ....
    }
```

- in this case the `_updateMultiDeposit` function is responsible for updating the deposit payload by resolves the final amounts given by the keeper and in this process the failed deposits will be removed from the payload body and set to `failedDeposit` to be Rescued later by the user.

```solidity
  function _updateMultiDeposit(
        uint256 payloadId_,
        bytes memory prevPayloadBody_,
        uint256[] calldata finalAmounts_
     )
        internal
        returns (bytes memory newPayloadBody_, PayloadState finalState_)
     {
        /// some code ...

        uint256 validLen;
        for (uint256 i; i < arrLen; ++i) {
            if (finalAmounts_[i] == 0) {
                revert Error.ZERO_AMOUNT();
            }
            // update the amounts :
            (multiVaultData.amounts[i],, validLen) = _updateAmount(
                dstSwapper,
                multiVaultData.hasDstSwaps[i],
                payloadId_,
                i,
                finalAmounts_[i],
                multiVaultData.superformIds[i],
                multiVaultData.amounts[i],
                multiVaultData.maxSlippages[i],
                finalState_,
                validLen
            );
        }
        // update the payload body and remove the failed deposits
        if (validLen != 0) {
            uint256[] memory finalSuperformIds = new uint256[](validLen);
            uint256[] memory finalAmounts = new uint256[](validLen);
            uint256[] memory maxSlippage = new uint256[](validLen);
            bool[] memory hasDstSwaps = new bool[](validLen);

            uint256 currLen;
            for (uint256 i; i < arrLen; ++i) {
                if (multiVaultData.amounts[i] != 0) {
                    finalSuperformIds[currLen] = multiVaultData.superformIds[i];
                    finalAmounts[currLen] = multiVaultData.amounts[i];
                    maxSlippage[currLen] = multiVaultData.maxSlippages[i];
                    hasDstSwaps[currLen] = multiVaultData.hasDstSwaps[i];
                    ++currLen;
                }
            }
            multiVaultData.amounts = finalAmounts;
            multiVaultData.superformIds = finalSuperformIds;
            multiVaultData.maxSlippages = maxSlippage;
            multiVaultData.hasDstSwaps = hasDstSwaps;
            finalState_ = PayloadState.UPDATED;
        } else {
            finalState_ = PayloadState.PROCESSED;
        }
        // return new payload
        newPayloadBody_ = abi.encode(multiVaultData);
    }
```

- An issue arises when some deposits fail, and others succeed. The function doesn't update the `retainERC4626` flags to match the new state.

```diff
    multiVaultData.amounts = finalAmounts;
    multiVaultData.superformIds = finalSuperformIds;
    multiVaultData.maxSlippages = maxSlippage;
    multiVaultData.hasDstSwaps = hasDstSwaps;
    finalState_ = PayloadState.UPDATED;
```

- This misalignment can lead to incorrect minting behavior in the `_multiDeposit` function, where the `retainERC4626` flags do not correspond to the correct `superFormsIds`.

- poc example :

- Bob creates a singleXchainMultiDeposit with Corresponding data and here's some:
  - `superFormsIds[1,2,3,4]`
  - `amounts[a,b,c,d]`
  - `retainERC4626[true,true,false,false]`
- After the cross-chain payload is received and all good, a keeper come and updates the amounts,assuming the update of amounts resulted in `a` and `b` failing, while `c` and `d` are resolved successfully.
- The `_updateMultiDeposit` function updates the payload to contain:
  - `superFormsIds[3,4]`
  - `amounts[c',d']`
  - `retainERC4626[true,true,false,false]`
- here the `retainERC4626` is not updated. in this case When the keeper processes the payload, `_multiDeposit` is triggered, bob is incorrectly minted superPositions for superForms 3 and 4, despite the user's preference to retain ERC-4626 shares for this superForms.

- Impact

- This issue can lead to incorrect minting or unminting behavior of superPosition, where users may receive `superPositions` when they intended to retain ERC-4626 shares, or vice versa.While it may be not a big issue for `EOAs` ,This can be particularly problematic for contracts integrating with Superform, potentially breaking invariants and causing loss of funds.

- Recommendation

The `retainERC4626` flags should be realigned with the updated `superFormsIds` and `amounts` to ensure consistent behavior. A new array for `retainERC4626` should be constructed in parallel with the other arrays to maintain the correct association.

Here is a suggested fix:

```solidity
// Inside the _updateMultiDeposit function
bool[] memory finalRetainERC4626 = new bool[](validLen);
// ... existing code to update superFormsIds and amounts ...

uint256 currLen;
for (uint256 i; i < arrLen; ++i) {
    if (multiVaultData.amounts[i] != 0) {
        // ... existing code to update finalSuperformIds and finalAmounts ...
        finalRetainERC4626[currLen] = multiVaultData.retainERC4626[i];
        ++currLen;
    }
}

// Update the multiVaultData with the new retainERC4626 array
multiVaultData.retainERC4626 = finalRetainERC4626;
```

### Attacker can deposit to non existing superform and change token that should be returned

**Severity:** Medium risk

**Context:** _(No context files were provided by the reviewer)_

- Proof of Concept
  To deposit on another chain user should [call `SuperformRouter.singleXChainSingleVaultDeposit`](https://github.com/superform-xyz/superform-core/blob/main/src/SuperformRouter.sol#L44) as option with single vault deposit. Then `BaseRouterImplementation._singleXChainSingleVaultDeposit` function will be called.

User provides `SingleXChainSingleVaultStateReq` param to the function were he can provide all info. One of such info is `superformId`, which is information about superform vault and chain where it is located.

`SuperformRouter.singleXChainSingleVaultDeposit` function then [validates that superfom data](https://github.com/superform-xyz/superform-core/blob/main/src/BaseRouterImplementation.sol#L135-L147). In case if it's direct deposit or withdraw(on same chain), then function [checks if superform exist](https://github.com/superform-xyz/superform-core/blob/main/src/BaseRouterImplementation.sol#L778-L780). In our case this check will be skipped as we deposit to another chain. Then it checks [that superform chain is same as destination chain](https://github.com/superform-xyz/superform-core/blob/main/src/BaseRouterImplementation.sol#L786). And after that there is a check, [that superform implementation is not paused](https://github.com/superform-xyz/superform-core/blob/main/src/BaseRouterImplementation.sol#L794), which is not important for us.
The function `_validateSuperformData` doesn't have ability to check if superform indeed exists on destination.

`_singleXChainSingleVaultDeposit` function doesn't have any additional checks for the superform, [so it packs superform](https://github.com/superform-xyz/superform-core/blob/main/src/BaseRouterImplementation.sol#L185) and [then send this to the destination](https://github.com/superform-xyz/superform-core/blob/main/src/BaseRouterImplementation.sol#L188-L200). This means that user can provide any superform id as deposit target.

In case of xChain deposits all funds should go to the dst swapper or core registry. This is forced by validators(both lifi and socket). Sp in case if you don't do dst swap, then funds will be bridged directly to the `CoreStateRegistry` on destination chain.

When message is relayed and proofs are relayed, then keepers can call `updateDepositPayload` and provide amount that was received from user after bridging. And only after that `processPayload` can be called. `updateDepositPayload` function [will call `_updateSingleDeposit`](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L101) to update amount that is received on destination chain, then [`_updateAmount` will be called](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L474-L485).

In case if there were no dstSwap then [superform is checked to be valid](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L561-L562) and also in case if slippage didn't pass, then code also goes into `if` block. So in our case, if we provided not valid superformId, then we go to the `if` block.

https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L561-L562

```solidity
                failedDeposits[payloadId_].superformIds.push(superformId_);

                address asset;
                try IBaseForm(_getSuperform(superformId_)).getVaultAsset() returns (address asset_) {
                    asset = asset_;
                } catch {
                    /// @dev if its error, we just consider asset as zero address
                }
                /// @dev if superform is invalid, try catch will fail and asset pushed is address (0)
                /// @notice this means that if a user tries to game the protocol with an invalid superformId, the funds
                /// bridged over that failed will be stuck here
                failedDeposits[payloadId_].settlementToken.push(asset);
                failedDeposits[payloadId_].settleFromDstSwapper.push(false);

                /// @dev sets amount to zero and will mark the payload as PROCESSED (overriding the previous memory
                /// settings)
                amount_ = 0;
                finalState_ = PayloadState.PROCESSED;
```

Inside this block payload is marked as failed and call to the superform is done to get asset in it. In case if vault has provided asset, then it will be set to the `failedDeposits[payloadId_].settlementToken`. As our superform vault is invalid and can be crafted by attacker it can be crafted in such way to provide more precious token than the one that was deposited by attacker.

Later failed payload will be resolved using `proposeRescueFailedDeposits` function and payload initiator will be refunded. And in case if they will not notice that token has changed, then it's possible that they will refund wrong amount of tokens. This is also a problem in case if attacker will compromise `CORE_STATE_REGISTRY_RESCUER_ROLE` role.

- Impact
  Attacker can get bigger amount of funds
- Recommended Mitigation Steps
  In case if superform is invalid, then don't update `failedDeposits[payloadId_].settlementToken.push(asset)` to the token returned by invalid vault.

### ERC4626FormImplementation.\_processDirectWithdraw sends funds to msg.sender instead of receiver

**Severity:** Medium risk

**Context:** _(No context files were provided by the reviewer)_

- Proof of Concept
  When user wants to do a withdraw on same chain(or on another chain), then he provides `receiverAddress` of the funds inside his request.

When request is executed in the `ERC4626FormImplementation._processDirectWithdraw` function, then this param is not used, but [`srcSender` is used instead](https://github.com/superform-xyz/superform-core/blob/main/src/forms/ERC4626FormImplementation.sol#L287). As result funds will be withdrawn to wrong address which can make problems, for example for some other vaults that are constructed upon superform vaults and should send funds to another recipient.

Pls, note that for withdraws on another chain, [funds are sent correctly](https://github.com/superform-xyz/superform-core/blob/main/src/forms/ERC4626FormImplementation.sol#L355C53-L355C69).

Why i believe this is high? Suppose that another contract uses superform as vault and holds all superpositions inside. Users receive shares when deposit into it. And in order to withdraw that contract will pass user's address as receiver. But funds will be sent back to the contract and it's really likely that user will get nothing. However for xChain it will work fine, so it's possible that they will not notice that quickly. As result such a contract will not be able to work normally and some user's will lose funds.

- Impact
  Funds can be sent to another recipient, which can lead to loss of funds.
- Recommended Mitigation Steps
  Send funds to the `receiverAddress` only. Also make sure [to use it here as well](https://github.com/superform-xyz/superform-core/blob/main/src/forms/ERC4626FormImplementation.sol#L317).

### Keeper will always overwrite the user `txData` in case of single crosschain withdraw

**Severity:** Medium risk

**Context:** _(No context files were provided by the reviewer)_

- summary :

- the keeper will always ovewrite the `txData` when updating the withdraw payload in case of a single withdraw , while it shouldn't be updated if the user already provided a `txData` like in multi withdraw behavior .

- Details :

- The [`updateWithdrawPayload`](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L115) function within the `CoreStateRegistry` contract plays the role of updating the withdrawal payload with a (`txData_`) provided by the keeper.Particularly in [\_updateWithdrawPayload](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L592) function.

```js
    // updateWithdrawPayload function
    // prev code ...
 >>    prevPayloadBody = _updateWithdrawPayload(prevPayloadBody, srcSender, srcChainId, txData_, isMulti);
    // more code ..
```

- This function is expected to call [\_updateTxData](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L640) to conditionally update the payload with the keeper's `txData_`.

```js
     // _updateWithdrawPayload() function
     // prev code ....
  >>  multiVaultData = _updateTxData(txData_, multiVaultData, srcSender_, srcChainId_, CHAIN_ID);
     // more code ..
```

- The [\_updateTxData](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L640) function should leave the user's `txData` unchanged if it is already provided (i.e., if its length is not zero).

```js
  function _updateTxData(/*....*/) internal view returns (InitMultiVaultData memory){

        uint256 len = multiVaultData_.liqData.length;

        for (uint256 i; i < len; ++i) {
   >>       if (txData_[i].length != 0 && multiVaultData_.liqData[i].txData.length == 0) {
             }
        }

        return multiVaultData_;
    }
```

- in case of singleWithdraw, regardless of whether the user's `txData` was provided or not, the function will always overwrite the user `singleVaultData.liqData.txData` to `txData_[0]` which is the keeper's data. instead of `txData` returned from [\_updateTxData](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L640) function . which will not be the keeper `txData` in case user provided data .

```js
  function _updateWithdrawPayload( bytes memory prevPayloadBody_, address srcSender_, uint64 srcChainId_,
    bytes[] calldata txData_,uint8 multi )internal  view returns (bytes memory){
       // prev code ..
        multiVaultData = _updateTxData(txData_, multiVaultData, srcSender_, srcChainId_, CHAIN_ID);

        if (multi == 0) {
            // @audit-issue : the keeper will always overwrite the txData,should be multiVaultData.liqData.txdata[0]
  >>        singleVaultData.liqData.txData = txData_[0];
            return abi.encode(singleVaultData);
        }

        return abi.encode(multiVaultData);
    }
```

- The correct behavior should ensure that the user's txData is preserved when provided (like in multi withdraw), and the keeper's txData is only utilized if the user's txData is absent.

- impact :

- The impact is somewhat unclear in this scenario because of the unknown keeper Behavior.however, due to the validation process before executing the `txData`, there are some potential issues i can think off In case `TxData` maliciously updated that will pass the validation:

1.  can set the `amountIn` for the swap too low, causing the withdrawn amounts of the user to swap only a small portion, with the remainder staying in the `superForm`.
2.  can alter the final received token after the swap to any token (e.g., swapping from USDC to WETH, the keeper might set it to swap from USDC to anyToken).
3.  can change the behavior of the `txData` (e.g : from swap and bridge, to only swap)

- Recommendation :

```Diff
// prev code ...
if (multi == 0) {
--          singleVaultData.liqData.txData = txData_[0];
++          singleVaultData.liqData.txData = multiVaultData.liqData.txdata[0]
            return abi.encode(singleVaultData);
        }
// more code ....

```

### Quorum assumed to be the same for all chains, could cause messages cannot be processed

**Severity:** Medium risk

**Context:** _(No context files were provided by the reviewer)_

**Description**:

When cross chain deposit/withdraw is requested via router, it will eventually trigger dispatch payload via core state registry and send the message trough user's provided ambs.

https://github.com/superform-xyz/superform-core/blob/main/src/BaseRouterImplementation.sol#L546-L548

```solidity
    function _dispatchAmbMessage(DispatchAMBMessageVars memory vars_) internal virtual {
        AMBMessage memory ambMessage = AMBMessage(
            DataLib.packTxInfo(
                uint8(vars_.txType),
                uint8(CallbackType.INIT),
                vars_.multiVaults,
                STATE_REGISTRY_TYPE,
                vars_.srcSender,
                vars_.srcChainId
            ),
            vars_.ambData
        );

        (uint256 fees, bytes memory extraData) = IPaymentHelper(superRegistry.getAddress(keccak256("PAYMENT_HELPER")))
            .calculateAMBData(vars_.dstChainId, vars_.ambIds, abi.encode(ambMessage));

        ISuperPositions(superRegistry.getAddress(keccak256("SUPER_POSITIONS"))).updateTxHistory(
            vars_.currentPayloadId, ambMessage.txInfo
        );

        /// @dev this call dispatches the message to the AMB bridge through dispatchPayload
>>>     IBaseStateRegistry(superRegistry.getAddress(keccak256("CORE_STATE_REGISTRY"))).dispatchPayload{ value: fees }(
            vars_.srcSender, vars_.ambIds, vars_.dstChainId, abi.encode(ambMessage), extraData
        );
    }
```

And before dispatching messages using the provided ambs, it first check if the amb count is passed the destination chain quorum value.

https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/BaseStateRegistry.sol#L155-L157

```solidity
    function _dispatchPayload(
        address srcSender_,
        uint8[] memory ambIds_,
        uint64 dstChainId_,
        bytes memory message_,
        bytes memory extraData_
    )
        internal
    {
        AMBMessage memory data = abi.decode(message_, (AMBMessage));
        uint256 len = ambIds_.length;

        if (len == 0) {
            revert Error.ZERO_AMB_ID_LENGTH();
        }

        /// @dev revert here if quorum requirements might fail on the remote chain
>>>     if (len - 1 < _getQuorum(dstChainId_)) {
            revert Error.INSUFFICIENT_QUORUM();
        }

        AMBExtraData memory d = abi.decode(extraData_, (AMBExtraData));

        _getAMBImpl(ambIds_[0]).dispatchPayload{ value: d.gasPerAMB[0] }(
            srcSender_,
            dstChainId_,
            abi.encode(AMBMessage(data.txInfo, abi.encode(ambIds_, data.params))),
            d.extraDataPerAMB[0]
        );

        if (len > 1) {
            data.params = message_.computeProofBytes();

            /// @dev i starts from 1 since 0 is primary amb id which dispatches the message itself
            for (uint8 i = 1; i < len; ++i) {
                if (ambIds_[i] == ambIds_[0]) {
                    revert Error.INVALID_PROOF_BRIDGE_ID();
                }

                if (i - 1 != 0 && ambIds_[i] <= ambIds_[i - 1]) {
                    revert Error.DUPLICATE_PROOF_BRIDGE_ID();
                }

                /// @dev proof is dispatched in the form of a payload
                _getAMBImpl(ambIds_[i]).dispatchPayload{ value: d.gasPerAMB[i] }(
                    srcSender_, dstChainId_, abi.encode(data), d.extraDataPerAMB[i]
                );
            }
        }
    }
```

Now, when message is received and need to be updated/processed, it will also check the quorum, but it uses source chain quorum.

https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L380-L381

```solidity
    function _getPayload(uint256 payloadId_)
        internal
        view
        returns (
            uint256 payloadHeader_,
            bytes memory payloadBody_,
            bytes32 payloadProof,
            uint8 txType,
            uint8 callbackType,
            uint8 isMulti,
            uint8 registryId,
            address srcSender,
            uint64 srcChainId
        )
    {
        payloadHeader_ = payloadHeader[payloadId_];
        payloadBody_ = payloadBody[payloadId_];
        payloadProof = AMBMessage(payloadHeader_, payloadBody_).computeProof();
        (txType, callbackType, isMulti, registryId, srcSender, srcChainId) = payloadHeader_.decodeTxInfo();

        /// @dev the number of valid proofs (quorum) must be equal or larger to the required messaging quorum
>>>     if (messageQuorum[payloadProof] < _getQuorum(srcChainId)) {
            revert Error.INSUFFICIENT_QUORUM();
        }
    }
```

This assumes source chain and destination chain have the same quorum, which is not always true, it is possible that amb is available in the src chain but not the dst chain or the other way around, making it possible for the quorum between chains is different. If this happened, the message quorum cannot be reached and message will stuck.

**Recommendation**:

Check which quorum is smaller between src chain and dst chain, and use the smaller value instead.

### Increasing quorum requirements will prevent messages from being processed

**Severity:** Medium risk

**Context:** _(No context files were provided by the reviewer)_

**Description**:

The admin can call `setRequiredMessagingQuorum` to set `requiredQuorum`.

```solidity
    function setRequiredMessagingQuorum(uint64 srcChainId_, uint256 quorum_) external override onlyProtocolAdmin {
        requiredQuorum[srcChainId_] = quorum_;

        emit QuorumSet(srcChainId_, quorum_);
    }
```

And the message can only be processed if the number of proof messages received is greater than the requiredQuorum.

```solidity
        if (messageQuorum[payloadProof] < _getQuorum(srcChainId)) {
            revert Error.INSUFFICIENT_QUORUM();
        }
```

The problem here is that when a cross-chain message is initiated, the length of its `ambIds` is fixed, i.e. the number of messages sent is fixed, which also means that the number of received proof messages, i.e. `messageQuorum`, will not exceed it.

If `setRequiredMessagingQuorum` is called to increase the `requiredQuorum` between the initiation and processing of a cross-chain message, then when the cross-chain message is processed, it will not be processed due to insufficient `messageQuorum`.

1.Consider chain A with requiredQuorum 3, a user initiates a cross-chain deposit with ambIds length 4.

2.When the message is successfully dispatched, admin calls setRequiredMessagingQuorum to increase the requiredQuorum to 5.

3.When the destination chain processes the message, the message will not be processed because the messageQuorum is at most 4 and less than 5.

**Recommendation**:

Consider caching the current quorum requirements in the payload when a cross-chain message is initiated, and using it instead of the latest quorum requirements for the checks

### Lack of User Control Over Share Allocation in Vault Deposits May Lead to Unpredictable Outcomes

**Severity:** Medium risk

**Context:** _(No context files were provided by the reviewer)_

- details :

- The protocol lacks validation to ensure that the number of shares minted by a vault upon deposit aligns with the user's expectations, and the user have no choice only to accept any amount of shares resulted when depositing to a vault. This is particularly concerning for vaults that implement additional deposit strategies, which could be susceptible to front-running or other market manipulations.

- For example, a vault might use the deposited assets to engage in yield farming activities, where the number of **shares** minted to the depositor is dependent on the current state of the external protocol. If this protocol's conditions change rapidly, such as through front-running,The current protocol design expose the user to front running attacks..
- example from the `_processDirectDeposit` function of a superForm:

```js
 function _processDirectDeposit(InitSingleVaultData memory singleVaultData_) internal returns (uint256 dstAmount) {
        // prev code ....
        if (singleVaultData_.retain4626) {
            //@audit : user can't refuse the amount of shares minted even if it's zero .
   >>       dstAmount = v.deposit(vars.assetDifference, singleVaultData_.receiverAddress);
        } else {
   >>         dstAmount = v.deposit(vars.assetDifference, address(this));
        }
    }
```

```js
function _directSingleDeposit( address srcSender_, bytes memory permit2data_,InitSingleVaultData memory vaultData_)
     internal virtual {
    //    prev code ...
        // @audit : the contract mint any amount resulted from depositing , and the user have no control of that
    >>    if (dstAmount != 0 && !vaultData_.retain4626) {
            /// @dev mint super positions at the end of the deposit action if user doesn't retain 4626
            ISuperPositions(superRegistry.getAddress(keccak256("SUPER_POSITIONS"))).mintSingle(
                srcSender_, vaultData_.superformId, dstAmount
            );
        }
    }
```

- impact :

- This could lead to users receiving fewer shares than anticipated if the vault's share price is affected by other on-chain activities.

- Recommendation :

- To mitigate this risk, the user should have the ability to specify his desired `mintedAmount` of shares when he depositing to a `superForm` .

### `FailedXChainDeposits` event should always emit in case of failed deposits even if there are fullfilments

**Severity:** Low risk

**Context:** _(No context files were provided by the reviewer)_

- Vulnerability details:

The [\_multiDeposit](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L750) function within the `CoreStateRegistry` contract is responsible for orchestrating deposits into multiple vaults. It handles the process by iterating over an array of superForm IDs and attempting to deposit the corresponding amounts.

- The function attempts to deposit into each specified superForm.
- Successful deposits set a `fulfilment` flag, while failed deposits set an `errors` flag.
- On `fulfilment`, the function returns with `_multiReturnData`.
- On `errors`, the function should emit a `FailedXChainDeposits` event for "CORE_STATE_REGISTRY_RESCUER_ROLE" keeper.

- An issue arises when there are both successful and failed deposits within the same transaction. The current implementation returns early when there is atleast one `fulfilment` (true), which can prevent the emission of the `FailedXChainDeposits` event for any simultaneous `errors`. This behavior lead to a scenario where the `"CORE_STATE_REGISTRY_RESCUER_ROLE" `keeper, who operates based on events, does not receive the signal to propose the failed deposit amounts.

```js
// ..prev code ....
    if (fulfilment) {
  >>     return _multiReturnData(
            srcSender_,
            multiVaultData.payloadId,
            TransactionType.DEPOSIT,
            CallbackType.RETURN,
            multiVaultData.superformIds,
            multiVaultData.amounts
        );
    }
    // @audit-issue early return prevents the following event from being emitted even if there are failed deposits
    if (errors) {
        emit FailedXChainDeposits(payloadId_);
    }
```

- The absence of the `FailedXChainDeposits` event due to early return in the case of `fulfilment` hinders the keeper ability to perform updates, assuming the payload was processed without issue Especially that the function [processPayload](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L147) will emit only `PayloadProcessed(payloadId_)` event. leaving the user unable to rescue their funds.

```js
function processPayload(uint256 payloadId_) external payable virtual override {
        // ... prev code ...
 >>    emit PayloadProcessed(payloadId_);
    }
```

- impact :

- keepers won't propose rescue amounts, leading to loss of user funds for failed deposits.

- Recommendation

- The [\_multiDeposit](https://github.com/superform-xyz/superform-core/blob/main/src/crosschain-data/extensions/CoreStateRegistry.sol#L750) function should be updated to emit `FailedXChainDeposits` event for any errors before handling successful deposits. This ensures all necessary events are emitted for keeper operations.

```js
if (errors) {
    emit FailedXChainDeposits(payloadId_);
}
if (fulfilment) {
    return _multiReturnData(
        // ... parameters ...
    );
}
```

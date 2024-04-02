# The Standard - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01. `SmartVaultV3`  Insufficient Slippage Protection](#H-01)
    - ### [H-02.  DoS  in `runLiquidation` Function Prevent liquidation Process](#H-02)
    - ### [H-03. SmartVaultV3  Mishandling Of Burn Fees](#H-03)
- ## Medium Risk Findings
    - ### [M-01. Removal Of an Accepted Token is Sandwichable](#M-01)
    - ### [M-02. SmartVaultV3 Potential Liquidation Blockage Due to Blacklisting ](#M-02)
    - ### [M-03. Reward Eligibility Issue Due to Incorrect Holder Removal](#M-03)
    - ### [M-04. Missing Staleness Checks In The `latestRoundData()` from Chainlink](#M-04)



# <a id='contest-summary'></a>Contest Summary

### Sponsor: The Standard

### Dates: Dec 27th, 2023 - Jan 10th, 2024

[See more contest details here](https://www.codehawks.com/contests/clql6lvyu0001mnje1xpqcuvl)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 3
   - Medium: 4
   - Low: 0


# High Risk Findings

## <a id='H-01'></a>H-01. `SmartVaultV3`  Insufficient Slippage Protection            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L211

## Summary
The `swap` function in `SmartVaultV3` lacks slippage protection for users,which expose users to sandwich attack  resulting in users receiving less than expected from trades.

## Vulnerability Details
- In `SmartVaultV3`, the `swap` function uses `calculateMinimumAmountOut` to determine the minimum amount of output tokens for a swap. However, if the user's collateral minus the swap amount is still enough to cover the minted `EUROs`, this function returns zero as the minimum amount out. This means there is no floor to the number of tokens the user should receive, leaving them vulnerable to market manipulation and sandwich attacks .
```js
    function calculateMinimumAmountOut(bytes32 _inTokenSymbol, bytes32 _outTokenSymbol, uint256 _amount)
        private
        view
        returns (uint256)
     {
        ISmartVaultManagerV3 _manager = ISmartVaultManagerV3(manager);
        uint256 requiredCollateralValue = minted * _manager.collateralRate() / _manager.HUNDRED_PC();
        uint256 collateralValueMinusSwapValue = euroCollateral() - calculator.tokenToEur(getToken(_inTokenSymbol), _amount);
        return collateralValueMinusSwapValue >= requiredCollateralValue
    >>      ? 0
            : calculator.eurToToken(getToken(_outTokenSymbol), requiredCollateralValue - collateralValueMinusSwapValue);
    }

    function swap(bytes32 _inToken, bytes32 _outToken, uint256 _amount) external onlyOwner {
        uint256 swapFee = _amount * ISmartVaultManagerV3(manager).swapFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        address inToken = getSwapAddressFor(_inToken);
   >>   uint256 minimumAmountOut = calculateMinimumAmountOut(_inToken, _outToken, _amount); 
        ISwapRouter.ExactInputSingleParams memory params = ISwapRouter.ExactInputSingleParams({
            tokenIn: inToken,
            tokenOut: getSwapAddressFor(_outToken),
            fee: 3000, 
            recipient: address(this), 
            deadline: block.timestamp,
            amountIn: _amount - swapFee,
   >>       amountOutMinimum: minimumAmountOut,
            sqrtPriceLimitX96: 0
        });
        inToken == ISmartVaultManagerV3(manager).weth()
            ? executeNativeSwapAndFee(params, swapFee)
            : executeERC20SwapAndFee(params, swapFee);
    }

    function setOwner(address _newOwner) external onlyVaultManager {
        owner = _newOwner;
    }
```

## Impact
Users can suffer financial losses due to receiving far fewer tokens than expected when swapping. T
## Tools Used
manual review
## Recommendations
- Modify the swap function to allow users to set their own `minimumAmountOut`, and then do check that the user given `minAmountOut` , not make the vault undercollateralised after the swap
## <a id='H-02'></a>H-02.  DoS  in `runLiquidation` Function Prevent liquidation Process            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPoolManager.sol#L59

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L205

## Summary
The `runLiquidation` function is at risk of a DoS attack due to the potential for the `holders` array to become excessively large, which can cause the `distributeAssets` function to consume too much gas and fail. This vulnerability can be exploited by repeatedly calling `increasePosition` with small amount to stake, leading to a halt in liquidations and significant financial damage to the protocol.
## Vulnerability Details

- The `runLiquidation()` function within the `LiquidationPoolManager` contract is tasked with the liquidation of a **smartVault** and distribution of assets got from the liquidated **smartVault** to stakers and protocol, proportional to their `TST` or `EURO` token staked in the pool. It achieves this by calling `distributeAssets()` function in `liquidationPool` which iterating over the `assets` for each `holder` in the `holders` arrays to calculate and allocate the rewards to each staker accordingly.
```js
  function runLiquidation(uint256 _tokenId) external {
        ISmartVaultManager manager = ISmartVaultManager(smartVaultManager);
        manager.liquidateVault(_tokenId); // this will return the Euros to this 
        distributeFees();// distribute the fees to all holders and pending stakers in the liquidation pool .
        ITokenManager.Token[] memory tokens = ITokenManager(manager.tokenManager()).getAcceptedTokens();
        ILiquidationPoolManager.Asset[] memory assets = new ILiquidationPoolManager.Asset[](tokens.length);
        uint256 ethBalance;
  >>      for (uint256 i = 0; i < tokens.length; i++) {
            ITokenManager.Token memory token = tokens[i];
            if (token.addr == address(0)) {
                ethBalance = address(this).balance;
                if (ethBalance > 0) assets[i] = ILiquidationPoolManager.Asset(token, ethBalance);
            } else {
                IERC20 ierc20 = IERC20(token.addr);
                uint256 erc20balance = ierc20.balanceOf(address(this));
                if (erc20balance > 0) {
                    assets[i] = ILiquidationPoolManager.Asset(token, erc20balance);
                    ierc20.approve(pool, erc20balance);
                }
            }
        }
        
  >>      LiquidationPool(pool).distributeAssets{value: ethBalance}(
            assets, manager.collateralRate(), manager.HUNDRED_PC()
        );
        forwardRemainingRewards(tokens);
    }
```
- The `runLiquidation` function within the manager contract is susceptible to a Denial of Service (DoS) attack due to the computational intensity of the `distributeAssets` function from the `LiquidationPool` contract that it invokes. 
```js
 function distributeAssets(
        ILiquidationPoolManager.Asset[] memory _assets,
        uint256 _collateralRate,
        uint256 _hundredPC
     ) external payable {
        consolidatePendingStakes();
        //@audit-ok no proper check for chainlink return data , see bellow also , espectialy for multiple assets
        (, int256 priceEurUsd,,,) = Chainlink.AggregatorV3Interface(eurUsd).latestRoundData();
 >>     uint256 stakeTotal = getStakeTotal();//loop1
        uint256 burnEuros;
        uint256 nativePurchased;
>>    for (uint256 j = 0; j < holders.length; j++) {//loop2
            Position memory _position = positions[holders[j]];
            uint256 _positionStake = stake(_position);
            if (_positionStake > 0) {
                for (uint256 i = 0; i < _assets.length; i++) {//loop3
                    // some code .....
            }
            positions[holders[j]] = _position;
        }
        if (burnEuros > 0) IEUROs(EUROs).burn(address(this), burnEuros);
        returnUnpurchasedNative(_assets, nativePurchased);//loop4
    }
```

- An attacker can inflate the `holders` array by calling `increasePosition()` with small stake values, exploiting the function's logic to add new,  This can lead to an oversized `holders` array, causing `distributeAssets` to fail due to excessive gas consumption when invoked by `runLiquidation`.
```js
  function increasePosition(uint256 _tstVal, uint256 _eurosVal) external {
        require(_tstVal > 0 || _eurosVal > 0);        
        consolidatePendingStakes();
        ILiquidationPoolManager(manager).distributeFees();
        if (_tstVal > 0) IERC20(TST).safeTransferFrom(msg.sender, address(this), _tstVal);
        if (_eurosVal > 0) IERC20(EUROs).safeTransferFrom(msg.sender, address(this), _eurosVal);
        pendingStakes.push(PendingStake(msg.sender, block.timestamp, _tstVal, _eurosVal));
 >>     addUniqueHolder(msg.sender);
    }
```
-  such an attack will halt  the liquidation system, resulting in financial losses and damaging the protocol significantly.

## Poc 
- here a poc that shows only with 1000 holder how many gas that can be consumed :
```js
 // SPDX-License-Identifier: SEE LICENSE IN LICENSE
pragma solidity ^0.8.17;

import "forge-std/console.sol";
import "forge-std/Test.sol";
import "../../contracts/utils/SmartVaultDeployerV3.sol";
import "../../contracts/SmartVaultManagerV5.sol";
import "../../contracts/utils/EUROsMock.sol";
import "../../contracts/utils/SmartVaultIndex.sol";
import "../../contracts/LiquidationPoolManager.sol";
import "../../contracts/LiquidationPool.sol";
import "../../contracts/SmartVaultV3.sol";
import "../../contracts/utils/ERC20Mock.sol";
import "../../contracts/utils/TokenManagerMock.sol";
import "../../contracts/utils/ChainlinkMock.sol";
 contract setup is Test {
    ChainlinkMock clmock;
    ChainlinkMock clmock1;
    TokenManagerMock tokenmanager;
    SmartVaultDeployerV3 deployer;
    SmartVaultManagerV5 manager;
    SmartVaultIndex vaultIndex;
    EUROsMock euro;
    LiquidationPoolManager poolmanager;
    LiquidationPool pool;
    ERC20Mock tst;
    ChainlinkMock clmock2;
    ERC20Mock token;
    address bob = makeAddr("bob");
    address alice  = makeAddr("alice");
     //////////////////////// standard function //////////////////////////////////
    function onERC721Received(address ,address ,uint ,bytes memory ) public pure returns (bytes4 retval){
        return this.onERC721Received.selector;
    }
    receive() external payable {}
    function setUp() public virtual {
        skip(3 days);
        // deploy chain link mock : 
        clmock = new ChainlinkMock("native price feed");
        clmock.setPrice(160000000000);
        clmock1 = new ChainlinkMock("euro price feed");
        clmock1.setPrice(150000000);
        // deploy the token manager : 
        tokenmanager = new TokenManagerMock(bytes32("native"),address (clmock));
         clmock2 = new ChainlinkMock("random token price feed");
         token = new ERC20Mock("token","tk",18);
         token.mint(bob,10000 ether);
         clmock2.setPrice(100000000);
         tokenmanager.addAcceptedToken(address(token),address(clmock2));
        // deploy smartVault manager : 
        manager = new SmartVaultManagerV5();
        // deploy smart index and set the setVaultManager(manager)
        vaultIndex = new SmartVaultIndex();
        vaultIndex.setVaultManager(address(manager));
        // deploy euro mock : 
        euro = new EUROsMock();
        // deploy tst : 
        tst = new ERC20Mock("test tst","tst", 18);
        // deploy smart vault deployer : and set the constaructor to (bytes32(native), address(this))
        deployer = new SmartVaultDeployerV3(bytes32("native"),address(clmock1));
        // deploy the liquidation pool : 
        // deploy the pool manager : 
        manager.initialize(vaultIndex,address(euro),address(tokenmanager));
        poolmanager = new LiquidationPoolManager(address(tst),address(euro),address(manager),address(clmock1), payable (address(this)),50000);
        pool = LiquidationPool(poolmanager.pool());
        // set the euro and index and deployer to the smart vault manager 
        manager.setSmartVaultDeployer(address(deployer));
        euro.grantRole(euro.DEFAULT_ADMIN_ROLE(),address(manager));
        manager.setLiquidatorAddress(address(poolmanager));
        manager.setMintFeeRate(10000);
        manager.setBurnFeeRate(5000);
        manager.setSwapFeeRate(5000);
        manager.setProtocolAddress(address(poolmanager));
        vm.deal(address(this), 20 ether);
    }

    //////////////////////////////////////////////////////////////////////////////

    /////////////////////////////////// liquidation poc ///////////////////////////
    
    function test_runLiquidation() public {
        // create a smart vault ;
        (address vault,uint tokenId) = manager.mint();
        // add collateral and mint euro ;
        (bool ok,) = vault.call{value: 2 ether}("");
        require(ok);
        SmartVaultV3(payable (vault)).mint(bob, 1600 ether);
        // make the price of collateral less to trigger undercollatirlization ;
         clmock.setPrice(1600000000);
        // create a huge number of holders  then skip two day (so the holders will);
        for (uint i;i< 1000;i++) {
            address to = address(uint160(uint(keccak256(abi.encode(i)))));
            tst.mint(to,i * 1 ether + 1);
            vm.startPrank(to);
            tst.approve(address(pool), tst.balanceOf(to));
            pool.increasePosition(tst.balanceOf(to),0);
            vm.stopPrank();
        }
        skip(3 days);
        // catch the gas before : 
        uint gasBefore = gasleft();
        poolmanager.runLiquidation(tokenId);
        uint gasConsumed = gasBefore - gasleft();
        console.log("gas used for runing liquidation with one token and 1000 holders is :",gasConsumed);
        console.log("number of holders ",pool.len());
        // try to run liquidation : 
    }
```
- console after runing test :
```sh
 Running 1 test for test/foundry_tests/setup.t.sol:setup
 [PASS] test_runLiquidation() (gas: 1798591326)
 Logs:
  gas used for runing liquidation with one token and 1000 holders is : 943865157
  number of holders  1000

 Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.99s
```
- notice that in this case the gas consumed is 93m , which too much ,and exceed the *block gas limit* for mainnet (for l2 the gas will be calculated differently ,however this can be targeted to make it exceed the block limit of any evm compatible due to large resource consuming )
## impact : 

-  Legitimate stakers are unable to receive their fees, undermining the staking pool's incentive structure.
- all super vaults will be unliquidatable , since the [runLiquidation](https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPoolManager.sol#L59) function will always run out of gas.


## Tools Used
vs code 
## Recommendations
- I would recommend to enable liquidation for **smartVaults** separately from Distributing assets to stakers of the in the liquidation pool.
## <a id='H-03'></a>H-03. SmartVaultV3  Mishandling Of Burn Fees            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L160

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L169

## Summary
The SmartVaultV3 contract's handling of burn fees in `EURO` tokens can lead to a systemic shortfall, preventing users from reclaiming their full collateral. This issue threatens the long-term viability of the protocol by locking in user funds and undermining trust.
## Vulnerability Details
- The `SmartVaultV3` contract demonstrates a flaw in its accounting for `minted` `EURO` tokens, which should reflect the vault's debt. Users minting `EURO` tokens are charged a mint **fee**, and when they burn `EURO` tokens to recover their collateral, they incur a burn **fee**. However, the **smartVault's** `burn` function decreases the `minted` variable solely by the net **amount of `EUROs` burned**, not accounting for the burn **fee**. This discrepancy leads to a mismatch between the total `EUROs` users expect to burn (the sum of **minted** variable in all vaults) and the actual `EUROs` available in circulation.

### Example Poc :
Consider a user (assume this is the first user for Simplicity) who creates a vault and deposits collateral to mint 10,000 `EUROs`. With a mint **fee** of 10%, the user ends up with 9,000 `EUROs`, while the protocol collects 1,000 `EUROs` as a **fee**. The situation is as follows:

- User's `EUROs`: 9,000
- Protocol's `EUROs` (mint **fee**): 1,000
- Total `EUROs` minted (recorded by the user's *smartVault*): 10,000
- `EUROs` `totalSupply`: 10,000
```js
     function mint(address _to, uint256 _amount) external onlyOwner ifNotLiquidated {
        uint256 fee = _amount * ISmartVaultManagerV3(manager).mintFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        require(fullyCollateralised(_amount + fee), UNDER_COLL);
        minted = minted + _amount + fee;
        EUROs.mint(_to, _amount);
        EUROs.mint(ISmartVaultManagerV3(manager).protocol(), fee);
        emit EUROsMinted(_to, _amount, fee);
    }
```

When the user aims to reclaim their collateral, they attempt to burn all their `EUROs`. Facing a burn **fee** of 5%, they need to burn 10,000 EUROs. Assuming the user obtains the 1,000 `EUROs` initially given to the protocol as mint **fees**, we have:

- User's `EUROs` to burn: 10,000
- Burn fee (5% of 10,000): 500 `EUROs`
```js
  function burn(uint256 _amount) external ifMinted(_amount) {
        uint256 fee = _amount * ISmartVaultManagerV3(manager).burnFeeRate() / ISmartVaultManagerV3(manager).HUNDRED_PC();
        minted = minted - _amount;
        EUROs.burn(msg.sender, _amount);
        IERC20(address(EUROs)).safeTransferFrom(msg.sender, ISmartVaultManagerV3(manager).protocol(), fee);
        emit EUROsBurned(_amount, fee);
    }
```

- For the user to burn their 10,000 `EUROs`, they must pay a **500 EUROs fee**. However, there are no additional `EUROs` in circulation to cover this fee since the total supply is precisely 10,000 `EUROs`, all in the user's possession. Consequently, the user cannot burn the entire **10,000 EUROs** due to the fee deficit.

- As users burn `EUROs` and pay fees, the total supply of `EUROs` reduces, but the recorded debt does not accurately decrease. With numerous users and transactions, the system will face a liquidity shortfall, lacking enough `EUROs` for users to pay off their debts and fully close their positions. This results in a system that is technically solvent but practically illiquid.

- For a synthetic asset like the `EURO` token, which aims to be stable, this logical inconsistency is particularly troubling. It undermines the stability and reliability of the `EURO` token, potentially preventing it from maintaining its peg and serving as a stable synthetic asset. If left unaddressed, this flaw could ultimately lead to the protocol's demise, as users lose confidence in the system's ability to maintain its core functionality,and because the vault is overcollateralized, users should always be able to close their positions and retrieve their collateral
## Impact
- Over the long term, the discrepancy in the SmartVaultV3 contract's fee accounting will result in a growing shortfall of EURO tokens. As more users attempt to close their positions, the cumulative effect of unpaid burn fees will increasingly prevent the full recovery of overcollateralized assets. This systemic issue can lead to significant capital being locked within the platform, undermining user confidence and the protocol's financial integrity.
## Tools Used
manual review 
## Recommendations
- To resolve the fee-related issues,i would recommend to use a separate asset (e.g., a native token, tst token) for the collection of burn fees. These fees can be directed to a dedicated system-controlled **smartVault** to mint euro tokens if needed,but distinct from individual user **smartVault**. This change ensures that the EURO token supply remains unaffected by fee deductions, allowing users to fully redeem their collateral from overcollateralized positions without encountering a token deficit.
		
# Medium Risk Findings

## <a id='M-01'></a>M-01. Removal Of an Accepted Token is Sandwichable            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L149

## Summary
The `SmartVaultV3` contract is susceptible to a sandwich attack when `tokenManager` contract removing collateral tokens. This can be exploited to mint an excessive amount of EURO tokens and withdraw the de-listed collateral, causing a huge financial damage to the protocol.

## Vulnerability Details
- The `SmartVaultV3` contract interacts with a `TokenManager` contract that maintains a list of accepted collateral tokens.which can be added or removed via [removeAcceptedToken](https://arbiscan.io/address/0x33c5A816382760b6E5fb50d8854a61b3383a32a0#code#F8#L45) function.

- The vulnerability arises from the fact that the [removeAsset](https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L149) function   in all [smartVaultsV3](https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L149) checks if the token being withdrawn is an accepted collateral token by calling getTokenIfExists on the TokenManager. If the token is not found, or if it is found but the vault is not undercollateralized after the removal, the withdrawal is allowed to proceed.

```js
 function removeAsset(address _tokenAddr, uint256 _amount, address _to) external onlyOwner {
    ITokenManager.Token memory token = getTokenManager().getTokenIfExists(_tokenAddr);
    if (token.addr == _tokenAddr) require(canRemoveCollateral(token, _amount), UNDER_COLL);
    IERC20(_tokenAddr).safeTransfer(_to, _amount);
    emit AssetRemoved(_tokenAddr, _amount, _to);
 }
```
- An attacker can exploit this by monitoring the mempool for transactions that indicate the removal of a token. They can then front-run this transaction by depositing a large amount of the token to be removed into their vault, minting huge amount of `EUROs` against it, and then back-run the transaction to withdraw the token once the removal is executed. This is possible because the token will be already removed .

- The lack of real-time collateral status checks during the minting process combined with the ability to withdraw tokens that are no longer accepted as collateral creates a window of opportunity for an attacker to execute a sandwich attack, minting `EUROs` with what is effectively non-collateral and then withdrawing the de-listed token, leaving the vault undercollateralized.

- another issue is that removing an asset can make even normal vault undercollatirlized. 
## poc : 
- here a poc that shows how the removing an accepted token can be sanwiched to mint a huge amount of **EUROs** and then withdraw the collateral used to mint this tokens, i used foundry with a custom setup for the protocol , i didn't use the proxy pattern so i have to set some intial values in the initialize function of smartVaultManager contract : 
```ts
 // SPDX-License-Identifier: SEE LICENSE IN LICENSE
 pragma solidity ^0.8.17;

 import "forge-std/console.sol";
 import "forge-std/Test.sol";
 import "../../contracts/utils/SmartVaultDeployerV3.sol";
 import "../../contracts/SmartVaultManagerV5.sol";
 import "../../contracts/utils/EUROsMock.sol";
 import "../../contracts/utils/SmartVaultIndex.sol";
 import "../../contracts/LiquidationPoolManager.sol";
 import "../../contracts/LiquidationPool.sol";
 import "../../contracts/SmartVaultV3.sol";
 import "../../contracts/utils/ERC20Mock.sol";
 import "../../contracts/utils/TokenManagerMock.sol";
 import "../../contracts/utils/ChainlinkMock.sol";
 contract setup is Test {
    ChainlinkMock clmock;
    ChainlinkMock clmock1;
    TokenManagerMock tokenmanager;
    SmartVaultDeployerV3 deployer;
    SmartVaultManagerV5 manager;
    SmartVaultIndex vaultIndex;
    EUROsMock euro;
    LiquidationPoolManager poolmanager;
    LiquidationPool pool;
    ERC20Mock tst;
    ChainlinkMock clmock2;
    ERC20Mock token;
    address bob = makeAddr("bob");
    address alice  = makeAddr("alice");
     //////////////////////// standard function //////////////////////////////////
    function onERC721Received(address ,address ,uint ,bytes memory ) public pure returns (bytes4 retval){
        return this.onERC721Received.selector;
    }
    receive() external payable {}
    function setUp() public virtual {
        skip(3 days);
        // deploy chain link mock : 
        clmock = new ChainlinkMock("native price feed");
        clmock.setPrice(160000000000);
        clmock1 = new ChainlinkMock("euro price feed");
        clmock1.setPrice(150000000);
        // deploy the token manager : 
        tokenmanager = new TokenManagerMock(bytes32("native"),address (clmock));
         clmock2 = new ChainlinkMock("random token price feed");
         token = new ERC20Mock("token","tk",18);
         token.mint(bob,10000 ether);
         clmock2.setPrice(100000000);
         tokenmanager.addAcceptedToken(address(token),address(clmock2));
        // deploy smartVault manager : 
        manager = new SmartVaultManagerV5();
        // deploy smart index and set the setVaultManager(manager)
        vaultIndex = new SmartVaultIndex();
        vaultIndex.setVaultManager(address(manager));
        // deploy euro mock : 
        euro = new EUROsMock();
        // deploy tst : 
        tst = new ERC20Mock("test tst","tst", 18);
        // deploy smart vault deployer : and set the constaructor to (bytes32(native), address(this))
        deployer = new SmartVaultDeployerV3(bytes32("native"),address(clmock1));
        // deploy the liquidation pool : 
        // deploy the pool manager : 
        manager.initialize(vaultIndex,address(euro),address(tokenmanager));
        poolmanager = new LiquidationPoolManager(address(tst),address(euro),address(manager),address(clmock1), payable (address(this)),50000);
        pool = LiquidationPool(poolmanager.pool());
        // set the euro and index and deployer to the smart vault manager 
        manager.setSmartVaultDeployer(address(deployer));
        euro.grantRole(euro.DEFAULT_ADMIN_ROLE(),address(manager));
        manager.setLiquidatorAddress(address(poolmanager));
        manager.setMintFeeRate(10000);
        manager.setBurnFeeRate(5000);
        manager.setSwapFeeRate(5000);
        manager.setProtocolAddress(address(poolmanager));
        vm.deal(address(this), 20 ether);
    }

     function test_removeAcceptedToken() public {
        uint initialBalance = token.balanceOf(bob);
        vm.prank(bob);
        (address vault,) = manager.mint();
        SmartVaultV3 v = SmartVaultV3(payable(vault));
        // bob sees that a remove token tx in the memepool so he front run it :
        // front run the tx that will remove this token
        uint euroMinted = frontrun(v);
        // remove the token 
        tokenmanager.removeAcceptedToken("tk");
        // backrun and withdraw the amount deposited to mint euro;
        backrun(v);
        assertTrue(v.undercollateralised());
        assertEq(token.balanceOf(bob),initialBalance);
        assertEq(euro.balanceOf(bob),euroMinted);
        
    }
    function frontrun(SmartVaultV3 v) internal returns(uint euroMinted){
        // send token to the vault : 
        
        vm.startPrank(bob);
        token.transfer(address(v),token.balanceOf(bob));
        // mint euro with this token : 
         euroMinted = v.status().maxMintable  *manager.HUNDRED_PC()  /manager.collateralRate();
        v.mint(bob,euroMinted);
        vm.stopPrank();

    }
    function backrun(SmartVaultV3 v) internal {
        // remove the asset : 
        vm.startPrank(bob);
        uint bal = token.balanceOf(address(v));
        v.removeAsset(address(token),bal,bob);
    }
}
```
## Impact

- The attacker obtaining an unwarranted amount of `EUROs` without collateral that can badly harm the protocol

## Tools Used
vs code 
manual review 
## Recommendations
- Implement a preventative measure to restrict user asset withdrawals if their vault becomes undercollateralized. 

```Diff
 function removeAsset(address _tokenAddr, uint256 _amount, address _to) external onlyOwner {
        ITokenManager.Token memory token = getTokenManager().getTokenIfExists(_tokenAddr);
        if (token.addr == _tokenAddr) require(canRemoveCollateral(token, _amount), UNDER_COLL);
        IERC20(_tokenAddr).safeTransfer(_to, _amount);
++        if(undercollateralised()) revert(UNDER_COLL);
        emit AssetRemoved(_tokenAddr, _amount, _to);
    }
```
## <a id='M-02'></a>M-02. SmartVaultV3 Potential Liquidation Blockage Due to Blacklisting             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L121

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/SmartVaultV3.sol#L110

## Summary
The `liquidate` function in `SmartVaultV3` can fail if any accepted token transfer reverts, potentially due to blacklisting, making the vault impossible to liquidate.

## Vulnerability Details
In `SmartVaultV3`, the `liquidate` function attempts to transfer all accepted tokens to the manager's address. If any token transfer fails, such as when a token uses a blacklist feature and the vault is blacklisted (possibly due to receiving tainted assets), the entire liquidation process will revert. This is because the function does not handle individual transfer rejections, leaving the vault in a state where it cannot be liquidated.

```js
function liquidate() external onlyVaultManager {
    require(undercollateralised(), "err-not-liquidatable");
    liquidated = true;
    minted = 0;
    liquidateNative();
    ITokenManager.Token[] memory tokens = getTokenManager().getAcceptedTokens();
    for (uint256 i = 0; i < tokens.length; i++) {
        if (tokens[i].symbol != NATIVE) liquidateERC20(IERC20(tokens[i].addr));
    }
}
```
- I don't know if that's currently possible with the blacklisting policy , but if a malicious actor can intentionally get the vault blacklisted by sending it a small amount of this asset( such as USDC which is known to have blacklisting capabilities and is an accepted token in the protocol) that will be blocklisting the vault, Once blacklisted, the vault becomes immune to the liquidation process, creating a significant risk for the protocol.


## Impact
- The potential for a vault to be blacklisted and thus become immune to liquidation poses a severe risk to the protocol. It allows a single point of failure to compromise the integrity of the liquidation process.
> `Note`: Considering the evolving regulatory environment in crypto, the EURO token's representation of fiat currency could attract regulatory actions, including blacklisting. This risk highlights the need to mitigate this for the protocol's compliance and stability.
## Tools Used
manual review 
## Recommendations
- For tokens that may exhibit unpredictable behavior, such as external ERC20 tokens with features like `blacklisting`, it is recommended to use a `try-catch` pattern within the liquidate function. This will isolate the failure of a single token transfer and allow the liquidation process to proceed with the remaining assets, ensuring that one token's failure does not impede the protocol's ability to liquidate a vault.
## <a id='M-03'></a>M-03. Reward Eligibility Issue Due to Incorrect Holder Removal            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L134

https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L149

## Summary
- A logic flaw in the `LiquidationPool` contract may cause users with pending stakes to become ineligible for rewards if they withdraw their entire staked amount before consolidation, due to premature removal from the `holders` array.
## Vulnerability Details

- When a user has both staked and pending stakes. The vulnerability is triggered under the following conditions:

1. A user already has a stake recorded in the `holders` array (e.g., `tst` = 1000, `euro` = 200).
2. The user decides to increase their stake, which results in additional amounts being added to `pendingStakes` (e.g., `tst` = 2000, `euro` = 100). The `consolidatePendingStakes()` function is designed to add unique holders to the `holders` array, so it does not add this user in this case because the user is already listed.
```js
    function increasePosition(uint256 _tstVal, uint256 _eurosVal) external {
        require(_tstVal > 0 || _eurosVal > 0); 
  >>    consolidatePendingStakes();
        ILiquidationPoolManager(manager).distributeFees();
        if (_tstVal > 0) IERC20(TST).safeTransferFrom(msg.sender, address(this), _tstVal);
        if (_eurosVal > 0) IERC20(EUROs).safeTransferFrom(msg.sender, address(this), _eurosVal);
  >>    pendingStakes.push(PendingStake(msg.sender, block.timestamp, _tstVal, _eurosVal));
        addUniqueHolder(msg.sender);
    }
```
3. If the user then decreases their entire staked position before the pending stakes are consolidated (within one day), the `empty()` check within the `decreasePosition()` function will lead to the user being removed from the `holders` array.
```js

   function decreasePosition(uint256 _tstVal, uint256 _eurosVal) external {
        consolidatePendingStakes();
        ILiquidationPoolManager(manager).distributeFees();// this will do something and callback to distributeRewared in this contract .. 
        require(_tstVal <= positions[msg.sender].TST && _eurosVal <= positions[msg.sender].EUROs, "invalid-decr-amount");// note non need for this
        if (_tstVal > 0) {
            IERC20(TST).safeTransfer(msg.sender, _tstVal);
            positions[msg.sender].TST -= _tstVal;
        }
        if (_eurosVal > 0) {
            IERC20(EUROs).safeTransfer(msg.sender, _eurosVal);
            positions[msg.sender].EUROs -= _eurosVal;
        }
   >>   if (empty(positions[msg.sender])) deletePosition(positions[msg.sender]);
    }

  function deletePosition(Position memory _position) private {
   >>   deleteHolder(_position.holder);
        delete positions[_position.holder];
    }
```
4. After the one-day period passes and `consolidatePendingStakes()` is called, the user's pending stakes are added to their position, but since they have been removed from the `holders` array, they are no longer eligible to get any rewards when `distributeFees()` or `distributeAssets` function called, cause in this case they won't be a holder (not in the **holders** array) nor they have a pending stake (it's already executed) even they are a real holder.

```js
 function distributeFees(uint256 _amount) external onlyManager {
        uint256 tstTotal = getTstTotal();
        if (tstTotal > 0) {
            IERC20(EUROs).safeTransferFrom(msg.sender, address(this), _amount);
    >>      for (uint256 i = 0; i < holders.length; i++) {
                address _holder = holders[i];
                positions[_holder].EUROs += _amount * positions[_holder].TST / tstTotal;
            }
    >>      for (uint256 i = 0; i < pendingStakes.length; i++) {
                pendingStakes[i].EUROs += _amount * pendingStakes[i].TST / tstTotal;
            }
        }
    }
```
## Poc : 
- here a poc that shows ,how bob can be deleted from array **holders** ,and not take any rewards , i used foundry with a custom setup for the protocol , i didn't use the proxy pattern so i have to set some intial values in the initialize function of smartVaultManager contract : 
```js
   // SPDX-License-Identifier: SEE LICENSE IN LICENSE
   pragma solidity ^0.8.17;

   import "forge-std/console.sol";
   import "forge-std/Test.sol";
   import "../../contracts/utils/SmartVaultDeployerV3.sol";
   import "../../contracts/SmartVaultManagerV5.sol";
   import "../../contracts/utils/EUROsMock.sol";
   import "../../contracts/utils/SmartVaultIndex.sol";
   import "../../contracts/LiquidationPoolManager.sol";
   import "../../contracts/LiquidationPool.sol";
   import "../../contracts/SmartVaultV3.sol";
   import "../../contracts/utils/ERC20Mock.sol";
   import "../../contracts/utils/TokenManagerMock.sol";
   import "../../contracts/utils/ChainlinkMock.sol";
   contract setup is Test {
      ChainlinkMock clmock;
      ChainlinkMock clmock1;
      TokenManagerMock tokenmanager;
      SmartVaultDeployerV3 deployer;
      SmartVaultManagerV5 manager;
      SmartVaultIndex vaultIndex;
      EUROsMock euro;
      LiquidationPoolManager poolmanager;
      LiquidationPool pool;
      ERC20Mock tst;
      ChainlinkMock clmock2;
      ERC20Mock token;
      address bob = makeAddr("bob");
      address alice  = makeAddr("alice");
      //////////////////////// standard function //////////////////////////////////
      function onERC721Received(address ,address ,uint ,bytes memory ) public pure returns (bytes4 retval){
         return this.onERC721Received.selector;
      }
      receive() external payable {}
      function setUp() public virtual {
         skip(3 days);
         // deploy chain link mock : 
         clmock = new ChainlinkMock("native price feed");
         clmock.setPrice(160000000000);
         clmock1 = new ChainlinkMock("euro price feed");
         clmock1.setPrice(150000000);
         // deploy the token manager : 
         tokenmanager = new TokenManagerMock(bytes32("native"),address (clmock));
            clmock2 = new ChainlinkMock("random token price feed");
            token = new ERC20Mock("token","tk",18);
            token.mint(bob,10000 ether);
            clmock2.setPrice(100000000);
            tokenmanager.addAcceptedToken(address(token),address(clmock2));
         // deploy smartVault manager : 
         manager = new SmartVaultManagerV5();
         // deploy smart index and set the setVaultManager(manager)
         vaultIndex = new SmartVaultIndex();
         vaultIndex.setVaultManager(address(manager));
         // deploy euro mock : 
         euro = new EUROsMock();
         // deploy tst : 
         tst = new ERC20Mock("test tst","tst", 18);
         // deploy smart vault deployer : and set the constaructor to (bytes32(native), address(this))
         deployer = new SmartVaultDeployerV3(bytes32("native"),address(clmock1));
         // intialize the custom manager  : 
         manager.initialize(vaultIndex,address(euro),address(tokenmanager));
           // deploy the pool manager : 
         poolmanager = new LiquidationPoolManager(address(tst),address(euro),address(manager),address(clmock1), payable (address(this)),50000);
         pool = LiquidationPool(poolmanager.pool());
         // set some initial values to manager : 
         manager.setSmartVaultDeployer(address(deployer));
         euro.grantRole(euro.DEFAULT_ADMIN_ROLE(),address(manager));
         manager.setLiquidatorAddress(address(poolmanager));
         manager.setMintFeeRate(10000);
         manager.setBurnFeeRate(5000);
         manager.setSwapFeeRate(5000);
         manager.setProtocolAddress(address(poolmanager));
         vm.deal(address(this), 20 ether);
      }
      function test_removedHolder() public {
         tst.mint(alice,100 ether);
         tst.mint(bob,1100 ether);
         vm.prank(bob);
         tst.approve(address(pool),type(uint).max);
         vm.prank(alice);
         tst.approve(address(pool),type(uint).max);
         // bob first stake some tst: 
         vm.prank(bob);
         pool.increasePosition(100 ether,0);
         vm.prank(alice);
         pool.increasePosition(50 ether,0);
         skip(2 days);
         // bob increase his position : 
         vm.startPrank(bob);
         pool.increasePosition(100 ether,0); 
         // before 1 day pass bob remove his first deposit :
            (LiquidationPool.Position memory bobPosition ,)= pool.position(bob);// bob position before rewards 
         (LiquidationPool.Position memory alicePosition ,)= pool.position(alice);// alice position before rewards
         pool.decreasePosition(100 ether,0);
         skip(2 days);vm.stopPrank();
         // lets say another user stakes which will trigger consolidatePendingStakes() (alice for simplicity)
         vm.prank(alice);
         pool.increasePosition(50 ether,0);
         // mint some euro to the LiquidationPoolManager , and distribute rewards
         euro.mint(address(poolmanager), 100 ether);
         poolmanager.distributeFees();
         (LiquidationPool.Position memory bobPositionAfter ,)= pool.position(bob);
         (LiquidationPool.Position memory alicePositionAfter ,)= pool.position(alice);
         console.log("- bob stakes before :",bobPosition.TST);
         console.log("- while alice stakes before is ", alicePosition.TST, " you can see that bob have more stakes then alice");
         console.log("- now bob got ",bobPositionAfter.EUROs," euro rewards ");
         console.log("- while alice had : ", alicePositionAfter.EUROs, " euro rewards ,even thougth bob have way more stake then her");     
      }
   }
```
- console after running test : 
```sh
  Running 1 test for test/foundry_tests/setup.t.sol:setup
  [PASS] test_removedHolder() (gas: 839224)
  Logs:
  - bob stakes before : 200000000000000000000
  - while alice stakes before is  50000000000000000000  you can see that bob have more stakes then alice
  - now bob got  0  euro rewards 
  - while alice had :  50000000000000000000  euro rewards ,even thougth bob have way more stake then her
```
## Impact
- - This flaw can result in a user being unfairly excluded from reward distributions, despite having a pending stake that should qualify them for rewards. 
## Tools Used
vs code 
## Recommendations

- It is recommended to remove the `addUniqueHolder()` function call from the `increasePosition()` function, as its current placement is unnecessary. When a holder is unique, it implies their position is empty, and thus they are not entitled to rewards at that moment. To ensure accurate tracking and reward eligibility, the `addUniqueHolder()` function should be called within the `consolidatePendingStakes()` function after pending stakes have been added to the holder's position.

```Diff
   // add addUniqueHolder();
  function consolidatePendingStakes() private {
        uint256 deadline = block.timestamp - 1 days;
        for (int256 i = 0; uint256(i) < pendingStakes.length; i++) {
            PendingStake memory _stake = pendingStakes[uint256(i)];
            if (_stake.createdAt < deadline) {
                positions[_stake.holder].holder = _stake.holder;//note no need for storing this .. 
                positions[_stake.holder].TST += _stake.TST;
                positions[_stake.holder].EUROs += _stake.EUROs;
                deletePendingStake(uint256(i));// @note : tooo heavy compitation .. 
++             addUniqueHolder(_stake.holder);
                i--;
            }
        }
    }

   // remove addUniqueHolder();
  function increasePosition(uint256 _tstVal, uint256 _eurosVal) external {
     //prev code ....
--        addUniqueHolder(msg.sender);
    }



```
## <a id='M-04'></a>M-04. Missing Staleness Checks In The `latestRoundData()` from Chainlink            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-12-the-standard/blob/main/contracts/LiquidationPool.sol#L207

## Summary
The `distributeAssets()` function lacks checks for stale or zero price data from Chainlink oracles, risking inaccurate distributions and blocking liquidations.
## Vulnerability Details

- The `distributeAssets()` function in the `LiquidationPool` contract uses Chainlink's `latestRoundData()` to fetch the latest price data for asset distribution calculations. However, the function does not perform checks to ensure that the price data is not stale. Stale data can result from various issues, such as oracle downtime, data source errors, or network congestion.

- Using stale data for price calculations can lead to incorrect distributions of assets to stakers, potentially causing halt of liquidation process in case zero eur/usd price return because the revert deu zero division  :

```js
(,int256 priceEurUsd,,,) = Chainlink.AggregatorV3Interface(eurUsd).latestRoundData();
(,int256 assetPriceUsd,,,) = Chainlink.AggregatorV3Interface(asset.token.clAddr).latestRoundData();
......................................................................
 uint256 costInEuros = _portion * 10 ** (18 - asset.token.dec) * uint256(assetPriceUsd)
 >>     / uint256(priceEurUsd) 
       * _hundredPC / _collateralRate; 
```
## Impact
Stale price data may cause incorrect asset distributions and, if `priceEurUsd` is zero, could halt liquidations due to transaction reversion.
## Tools Used
manual review
## Recommendations

It is recommended to use Chainlink’s latestRoundData() function with
checks on the return data  for example:

```js
 ( uint80 roundId , int256 price , uint256 startedAt , uint256updatedAt , uint80 answeredInRound ) = Chainlink.AggregatorV3Interface(eurUsd).latestRoundData();
require ( answeredInRound >= roundID , " Stale price ");
require ( price > 0, " invalid price ");
require ( block . timestamp <= updatedAt + 1 hour , " Staleprice ");
```





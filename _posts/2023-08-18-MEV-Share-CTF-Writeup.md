---
title: 🤝 MEV-Share CTF Writeup
author: Ashiq Amien
date: 2023-08-18 15:50:00 +0200
categories: [CTF, Flashbots]
tags: ctf
---

[Flashbots](https://www.flashbots.net/) hosted the [MEV-Share CTF](https://ctf.flashbots.net/), a set of challenges based around their MEV-Share product. The CTF ran from 5 to 7 Aug, but I didn't have much free time and wasn't able to get running in the few hours I had. I noticed the CTF infrastructure was up again this week and decided to have a bash while it was still running. Besides taking a hint to get up and running, I managed to solve all of the challenges and I've written up my solutions here :)


## Intro + setup

MEV-share is a protocol that allows searchers to backrun user transactions, while redistributing some of the profits back to the user. The CTF involves backrunning transactions from the CTF owner, with criteria specified in a smart contract. That is, a transaction eventually needs to call [`registerCapture()`](https://goerli.etherscan.io/address/0x6c9c151642c0ba512de540bd007afa70be2f1312#code#F1#L49), and the contracts to do that are found in [this transaction's calldata](https://goerli.etherscan.io/tx/0x20ad7a3656f0bf39d977400cf9dec21e8da1b238d48f2c1c850559a5948b5780). To know which transactions to backrun, there is an [event stream](https://docs.flashbots.net/flashbots-mev-share/searchers/event-stream) to listen, and you can [compose](https://docs.flashbots.net/flashbots-mev-share/searchers/understanding-bundles) and [send bundles](https://docs.flashbots.net/flashbots-mev-share/searchers/sending-bundles) based on the transaction information that you get.

To get started, I tried to use the [examples](https://github.com/flashbots/mev-share-client-ts/tree/main/src/examples) from `mev-share-client-ts`. I wasn't able to get this working for some reason- I ended up rabbit-holing on how mutexes work, trying to rewrite the script from scratch, and just doing a bunch of other debugging. After wasting enough time, I took inspiration from the [winning writeup](https://github.com/minaminao/ctf-blockchain/tree/main/src/MEVShareCTF) to get started and eventually landed on [this simple script](https://gist.github.com/AshiqAmien/4de77ea4b78a0542370d5cf5388ceb12) to start backrunning txs. The goal of the all of the challenges is to backrun an `activate*` function and meet the specified conditions. 

### Flags 1, 2, and 3

The smart contracts for the first three challenges were all the same, with the same `claimReward()` function to call in the backrun tx:

```typescript
function activateRewardSimple() external payable onlyOwner {
    activeBlock = block.number;
    emit Activate();
}

function claimReward() external {
    require (activeBlock == block.number);
    activeBlock = 0;
    mevShareCaptureLogger.registerCapture(captureId, tx.origin);
}
```

The challenge differences between these flags were in how they showed up in the event stream. To figure out when the transaction being sent to the contract we're interested in, you can perform a check with: 

```bash
$ curl https://mev-share-goerli.flashbots.net --stderr - | grep -i '<address>'
```

In this case, the first challenge is at address `0x98997b55Bb271e254BEC8B85763480719DaB0E53`, and we get the following output:

```bash
data: {"hash":"0x0fe3f9281638b6c11fe28a665da248fa37681d3b91a415af303164b9a82a6a65","logs":[{"address":"0x98997b55bb271e254bec8b85763480719dab0e53","topics":["0x59d3ce47d6ad6c6003cef97d136155b29d88653eb355c8bed6e03fbf694570ca"],"data":"0x"}],"txs":null,"mevGasPrice":"0x2faf080","gasUsed":"0x7530"}
```

To catch transactions going to this address, our `if` statement when listening for events in the solution script should be: 

```bash
if(tx?.logs?.[0].address === '0x98997b55Bb271e254BEC8B85763480719DaB0E53'){...
```

To call `claimReward()` to capture the flag, we need to specify the data field to just be the function selector `0xb88a802f` - there are no params to worry about in this case. Constructing the bundle as the following gets us the flag:

```typescript
const bundle = [
    { hash: tx.hash },
    { tx: await wallet.signTransaction(unsignedtx), canRevert: false },
]
```

Flags 2 and 3 were similar, with the address field populated instead of being in the logs. The only difference in the script would be to modify the `if` statement as follows:

```bash
if (tx?.to === "0x65459dd36b03af9635c06bad1930db660b968278".toLocaleLowerCase()) {...
```

### Flag 4

Flag 4 had the same smart contract as flags 1-3, but there was a difference in the challenge. Running the following provided no output, yet transactions were still landing on-chain:

```bash
$ curl https://mev-share-goerli.flashbots.net --stderr - | grep -i '0x20a1A5857fDff817aa1BD8097027a841D4969AA5'
```

To make sure transactions were still coming through the event stream, I piped the stream output and watched until a fresh `activateRewardSimple()` transaction landed on-chain. 

![landed bundle](/assets/img/sample/mevshare.png)

I then matched it to a tx in the piped contents:

```bash
data: {"hash":"0xc490ddd3bd34a9ffe2d60f8fadc4c57652e7849fa6403a8a2d10453da0c33823","logs":null,"txs":null,"mevGasPrice":"0x2faf080","gasUsed":"0x7530"}
```

This meant the transactions were coming through the event stream and landing on-chain, just with no further data to the searcher. This means that I needed to just try to backrun any txs with no logs. To get the flag, my filter was simply:

```bash
if (!tx?.logs) {...
```

### Flags 5, 6 and 7

Flags 5, 6 and 7 were different: the user needed to guess the correct magic number in the same block as the tx that set the number. The description of the challenges are as follows:

- The magic number for each flag lies in some range of 40 numbers, with the range boundary given in the logs from the event stream.
- Flag 6 has an additional constraint of requiring `tx.origin == msg.sender`
- Flag 7 has the constraint of flag 6, as well as that there are no retries for an incorrect guess. Each `tx.origin` only gets one chance. 

The code for flag 7 is below:

```typescript
function claimReward(uint256 _magicNumber) external {
    require(tx.origin == msg.sender);
    require(registeredV3Attempts[tx.origin] == false);
    registeredV3Attempts[tx.origin] = true;
    claimRewardInternal(_magicNumber, 203);
}

function claimRewardInternal(uint256 _magicNumber, uint256 _captureId) internal returns (bool) {
    if (activeBlock != block.number || _magicNumber != magicNumber) {
        return false;
    }
    activeBlock = 0;
    magicNumber = 0;
    mevShareCaptureLogger.registerCapture(_captureId, tx.origin);
    return true;
}
```

Solving flag 5 is easy: the backrun should be a call to a custom contract that does a for-loop to call the `claimReward()` function over all the possible magic numbers. However, this wouldn't work in flags 6 and 7 because of the `tx.origin == msg.sender` requirement. I noticed that the challenges stack quite nicely, so solving flag 7 would be a valid soluton for flags 5 and 6. Since I was pressed for time, this is what I did in the end. 

My first thought was that since we can nest bundles together, we could nest a bunch of bundles inside each other so that we hit each possible magic number with each transaction. But since this was only halfway through the challenge, this seemed too complicated, and I eventually just settled on spamming 40 bundles with each backrun being a different possible magic number. This worked for each flag :) 

```typescript
let args = magicV1.interface.parseLog({topics: [topics[0]], data: data});
let lowerBound = args?.args[0];
let upperBound = args?.args[1]; 

for(let i = 0; i<40; i++){                    
    const backrunPartial = await magicV1.claimReward.populateTransaction((BigInt(lowerBound) + BigInt(i)));
    const unsignedtx = {
        ...backrunPartial,
        chainId: provider._network.chainId,
        nonce: await wallet.getNonce(),                    
        gasLimit: 400000,                    
        maxFeePerGas: toBigInt(feeData.maxFeePerGas || 42) + BigInt(1e3),
        maxPriorityFeePerGas: toBigInt(feeData.maxPriorityFeePerGas || 2) + BigInt(1e3),
    }                

    const bundle = [
        { hash: tx.hash },
        { tx: await wallet.signTransaction(unsignedtx), canRevert: false },
    ]
    const targetBlock = await provider.getBlockNumber() + 1;
    const bundleParams: BundleParams = {
        inclusion: {
            block: targetBlock,
            maxBlock: targetBlock - 1 + NUM_TARGET_BLOCKS,
        },
        body: bundle,
    }
...
```

### Flags 8 and 9

Flags 8 and 9 were in the same contract, with two different activate functions to backrun:

```typescript
function activateRewardNewContract(bytes32 salt) external payable onlyOwner {
    MevShareCTFNewContract newlyDroppedContract = new MevShareCTFNewContract{salt: salt}();
    childContracts[address(newlyDroppedContract)] = 1;
    emit Activate(address(newlyDroppedContract));
}

function activateRewardBySalt(bytes32 salt) external payable onlyOwner {
    MevShareCTFNewContract newlyDroppedContract = new MevShareCTFNewContract{salt: salt}();
    childContracts[address(newlyDroppedContract)] = 2;
    emit ActivateBySalt(salt);
}
```
Both functions create new `MevShareCTFNewContract` contracts, and the goal is to backrun the relevant activate tx and call `claimReward()` in the new contract. To backrun a call to `activateRewardNewContract()`, we just need to grab the logs from the `Activate` event. Since both flags are on this contract, we can't just filter by address or we'll have to sift between `Activate` and `ActivateBySalt` events. Instead, since the event topic is emitted, we can filter by that:

```typescript
if ((tx?.logs)?.[0].topics[0] === targetTopic) {
    let data = tx?.logs?.[0].data;
    let args = targetContract.interface.parseLog({ topics: [topics[0]], data: data });
    let emittedAddress = args?.args[0];
    ...
```

With the address we can construct our bundle as usual, with the target address as the `emittedAddress`. This gets us flag 8. Flag 9 also creates a `MevShareCTFNewContract` contract, but the address is not emitted, only the salt. [With just the salt we can figure out what the address of the new contract](https://ethereum.stackexchange.com/questions/760/how-is-the-address-of-an-ethereum-contract-computed) is using ethers' `getCreate2Address` as follows:

```typescript
let saltedAddy = getCreate2Address(targetAddress, salt, bytecodetargethash); 
```

`TargetAddress` is the address of the contract performing the creation and `bytecodetargethash` is the init bytecode. We can find this init bytecode by [poking around on etherscan](https://goerli.etherscan.io/vmtrace?txhash=0xb19ca314a5f3bd986c5d3fc79c1d8accb14f69125b513686c07a14471e955234&type=parity#raw). As a sanity check, you can test that your salted addresses are being calculated correctly by using the salt and the outcome of an [`activateRewardNewContract` tx](https://goerli.etherscan.io/tx/0xb19ca314a5f3bd986c5d3fc79c1d8accb14f69125b513686c07a14471e955234/advanced). The following gets us the flag:

```typescript
if ((tx?.logs)?.[0].topics[0] === targetTopic) {
    console.log("found tx: " + tx.hash);
    const feeData = await provider.getFeeData();            
    let topics = tx?.logs?.[0].topics;
    let data = tx?.logs?.[0].data;
    let args = targetContract.interface.parseLog({ topics: [topics[0]], data: data });
    let salt = args?.args[0];                        
    const bytecodetarget = '0x60a06....'
    const bytecodetargethash = keccak256(bytecodetarget);
    let saltedAddy = getCreate2Address(targetAddress, salt, bytecodetargethash);  
    console.log("this is the salted addy: " + saltedAddy);          
    const callthisone = new Contract(saltedAddy, createdTargetABI, wallet);

    const backrunPartial = await callthisone.claimReward.populateTransaction();
    const unsignedtx = {
        ...backrunPartial,
        chainId: provider._network.chainId,
        nonce: await wallet.getNonce(),
        gasLimit: 400000,
        maxFeePerGas: toBigInt(feeData.maxFeePerGas || 42) + BigInt(1e3),
        maxPriorityFeePerGas: toBigInt(feeData.maxPriorityFeePerGas || 2) + BigInt(1e3),
    }

    const bundle = [
        { hash: tx.hash },
        { tx: await wallet.signTransaction(unsignedtx), canRevert: false },
    ]
```

### Flag 10

The final flag simply requires three of the same backrun tx's in the same block:

```typescript
function claimReward() external {
    require (activeBlock == block.number);
    require (tx.origin == msg.sender);
    uint256 claimCount = addressBlockCount[tx.origin][block.number] + 1;
    if (claimCount == 3) {
        mevShareCaptureLogger.registerCapture(401, tx.origin);
        return;
    }
    addressBlockCount[tx.origin][block.number] = claimCount;
}
```
Looking at how bundles were explain in [the docs](https://docs.flashbots.net/flashbots-mev-share/searchers/understanding-bundles), it wasn't very clear that include several txs directly in a bundle. I figured I need to compose my bundles so that the nested bundle does all three backruns. After many many attempts, I just couldn't seem to get the structure of my nested bundle to work ([even with `simulateBundle`](https://docs.flashbots.net/flashbots-mev-share/searchers/debugging)). By some random combination of attempts, I noticed a [singular tx bundle went through](https://goerli.etherscan.io/tx/0x1723c1254fac07686412212a57160234f16d487f18eeb4787de11b54a3fda3ad), and was infront of the `activateRewardTriple()` call in the block. This was unintuitive, I didn't think that sending a single tx without a backrun would work. I further saw that I was getting the gas price (`mevGasPrice`) of the incoming tx and realized I could manually set my gas price to backrun the tx:

```bash
$curl https://mev-share-goerli.flashbots.net --stderr - | grep -i '0x1eA6Fb65BAb1f405f8Bdb26D163e6984B9108478'

data: {"hash":"0xfd603317042fea7b534bf06ec2a8be362c1463475317aff1a299d8ce00805aa5","logs":[{"address":"0x1ea6fb65bab1f405f8bdb26d163e6984b9108478","topics":["0x59d3ce47d6ad6c6003cef97d136155b29d88653eb355c8bed6e03fbf694570ca"],"data":"0x"}],"txs":null,"mevGasPrice":"0x2faf080","gasUsed":"0x7530"}
```

Since the gas price was `0x2faf080`, I manually constructed 3 txs that called `claimReward()` with gas price `0x2faf079`. I realized that `0x2faf080 - 0x1` is NOT `0x2faf079` but rather `0x2faf07f`. I'll blame my 2am brain for that. 

This is my final solution: it's clunky, and makes me think it was unintended after reading the solutions of other users. But it worked!

```typescript
const backrunPartial = await targetContract.claimReward.populateTransaction();
const unsignedtx = {
    ...backrunPartial,
    chainId: provider._network.chainId,
    nonce: (await wallet.getNonce()) ,
    gasLimit: 400000,
    maxFeePerGas: 0x2faf079,
    maxPriorityFeePerGas: 0x2faf078
}
const unsignedtx2 = {
    ...backrunPartial,
    chainId: provider._network.chainId,
    nonce: (await wallet.getNonce()) + 1 ,
    gasLimit: 400000,
    maxFeePerGas: 0x2faf079,
    maxPriorityFeePerGas: 0x2faf078
}
const unsignedtx3 = {
    ...backrunPartial,
    chainId: provider._network.chainId,
    nonce: (await wallet.getNonce()) + 2 ,
    gasLimit: 400000,
    maxFeePerGas: 0x2faf079,
    maxPriorityFeePerGas: 0x2faf078
}

const targetBlock = await provider.getBlockNumber() + 1;

const bundle = [                                                
    { tx: await wallet.signTransaction(unsignedtx), canRevert: true },                 
    { tx: await wallet.signTransaction(unsignedtx2), canRevert: true },                 
    { tx: await wallet.signTransaction(unsignedtx3), canRevert: true },                 
]                                               
const childparams: BundleParams = {
    inclusion: {
        block: targetBlock,
        maxBlock: targetBlock - 1 + NUM_TARGET_BLOCKS,
    },
    body: bundle                                                    
}               
const childbundleResult = await mevShareClient.sendBundle(childparams);
```

#### Side note

I thought the way I solved flag 10 introduced a security issue for the protocol since if we know the gas price, we'd be able to frontrun transactions, which is voilation of the promise of using the protocol. But I double checked, mainnet event streams don't include the gas price and therefore is not an issue :)

### Final remarks

Overall, there was a bunch of things that went wrong for me in this CTF:

- rabbit holing how async programming works - not needed at all in the real answer, but the example from the ts client repo was my starting point 
- leaving out the simulation results and not being able to debug properly
- not being super familiar with typescript made me slower 
- long wait times waiting for flashbots to land a block on goerli, making me think I was doing something wrong and restarting my script
- not understanding why my script wasn't catching events (it just timed out, but I probs should have made it wait longer while listening for events)

These are all good things to reflect on. But overall, this was a really cool CTF and I really enjoyed it. It was a great way to dip my toes into MEV-share :)



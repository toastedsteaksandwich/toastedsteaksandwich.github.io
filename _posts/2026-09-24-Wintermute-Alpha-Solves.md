---
title: 🐎 Wintermute Alpha 2026 Writeups 
author: Ashiq Amien
date: 2026-09-24 12:45:00 +0200
categories: [CTF, MEV]
tags: ctf
math: true
---

Wintermute released their [2026 Alpha challenges](https://alpha.wintermute.com/case-studies) and this blog post details the solves of each one. Unlike previous Alpha challenges, there was no time limit, no leaderboard and no prizes to solve them. AI models render honest CTF competition nearly useless as-is today, so I believe this is why Wintermute decided to release these challenges as a self-study. 

Like all of my other CTF writeups, I try to describe my solves with at least as much detail as necessary, along with some of the common rabbit holes I fell into to show some of my thought process behind it. Overall, I used AI for mundane tasks such as generating or debugging basic syntax, or otherwise in a way that helps me understand the requirements of the challenge, but not to adjust the essence of the challenge/learning curve behind it. 

## Chal 1 - Out of Nowhere

This challenge gives us a [bridge unlock tx](https://etherscan.io/tx/0xe7b8d46c3f3e5f727cb42c9dfe7fc36855ab5092cf160e4c8812a2a27a84350b) and we need to find the originating tx on the other chain. The first clue is seeing that the tx has a `lockSource` parameter `0x53544b5a` which decodes to STKZ. My immediate intuition was that this is Stacks, but the Allbridge documentation was surprisingly unhelpful to confirm that this was the case. I tried researching a few things like:

- Look at the Allbridge contracts on Stacks [here](https://explorer.hiro.so/txid/SPN9GJ5CK3F3X4NPR7FNEQ72TAAHA0EGB9A7DEG5.bridge2?chain=mainnet) and [here](https://explorer.hiro.so/txid/0x3291e060bce97a80825b931ced3b5f21cc8550660e905686c6878ade0f518972?chain=mainnet&tab=transactions) 
- Read the [decompiled Validator contract](https://app.dedaub.com/ethereum/address/0x93746538d4519c809827205bd1c2c7a0e15bd74b/decompiled?line=62) to see if there was any extra info around the way the `lockSource` param was used  
- Trace all the [tx's on evm chains with Blockscan](https://blockscan.com/address/0xEc5f2EFa1A13c81179dDb0f0d4385e99E275994b#transactions) around the unlock tx time
- Check the tx on the allbridge/crosschain explorers [here](https://core.allbridge.io/explorer?search=0xe7b8d46c3f3e5f727cb42c9dfe7fc36855ab5092cf160e4c8812a2a27a84350b) and [here](https://explorer.range.org/transactions?s=0xe7b8d46c3f3e5f727cb42c9dfe7fc36855ab5092cf160e4c8812a2a27a84350b)

It turned out that these rabbit holes came about because I was reading Allbridge Core material, not Allbridge Classic. The [actual repo](https://github.com/allbridge-public/allbridge-contract-docs/tree/master) has more info and explains that the `lockSource` is the encoded blockchain ID. Even in this README, it doesn't show that STKZ/STX is supported. 

Fortunately, the README also describes a [token-info endpoint](https://allbridgeapi.net/token-info) that has a comprehensive list of supported chains and corresponding tokens. Here, we can find the actual Allbridge classic bridge contract `0x59743b04043e82a9057686e99e8f7503adf78013b68d46e6eb32641450594d62`. Tracing through txs around the time of the unlock tx age we can find the [corresponding lock tx](https://explorer.hiro.so/txid/0x36f2d5c245d08de980d0d23e4bd23b088312ce9e4b9845b4fd71930f52aab8fc?chain=mainnet).

![lock](/assets/img/sample/wintermute/stacks.png)

## Chal 2 - Falling Dutchman

In this challenge there seems to be an inefficiency in a Dutch auction that allows us to turn 0.1 ETH into 4 ETH. So how does it work? DutchX has two phases (for each token pair):

- The batching before an auction starts (for sellers to deposit their tokens)
- The actual running Dutch auction (when bidders are active).

The documentation for this protocol is unbelievably vague, but I was happy to find [this doc](https://dutchx.readthedocs.io/en/latest/_downloads/a33647e2481d5672e22d56a13c6ed54d/DutchX_Documentation.pdf) which was much clearer. In any case, to start, I need to figure out what markets/auctions are running. I didn't find a lazy way to enumerate all tokens, so I just poked at the [historical txs around the block time](https://etherscan.io/tokentxns?a=0xb9812e2fa995ec53b5b6df34d21f9304762c5497&ps=100&p=2) and just started enumerating some of them. Since this is meant to be a basic challenge, I just started with a select few tokens (OWL, DAI, SAI, GNO, KNC, RDN, OMG) since I don't think it's meant to be crazy.

I used the [helper contract](https://etherscan.io/address/0x64832950AbCcaa3d02aB8eB09AA058d789f5bB6a) and got some running pairs with `getRunningTokenPairs()`. From here I got the token price with `getCurrentAuctionPrice()`, which needs an `auctionIndex` provided by the `getAuctionStart()` method. This gives us price numerators and denominators. I saw GNO/WETH had a 0 numerator and non-zero denominator, so I thought this was a token priced at 0 that we can scoop up, but this turned out to be a dead end after some hours. It turned out KNC was the affected token, and rolling the chain forward we can see that the price goes to 0, so it meant we were in the [last 24 hours of the auction because the price reached 0](https://ethresear.ch/t/dutchx-fully-decentralized-auction-based-exchange/2443). 

Further, we know this is the affected token because we can use `getPriceOfTokenInLastAuction()` to see that the price of KNC in the previous auction was:

```java
token1 0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2
token2 0xdd974D5C2e2928deA5F71b9825b8b646686BD200
auctionIndex: 1051
price num: 27066572176327751486250
price den: 737680162678520003884077975

> 0.00158902304
```

while the current KNC price at the time was `0.00003669147`, a ~40x difference, which is inline with the challenge description. From here, simply `deposit WETH -> call postBuyOrder() -> claim+withdraw -> sell on Uniswap`:

```java
IWETH(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2).deposit{value: 0.1 ether}();
IWETH(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2).approve(address(dx), 0.1 ether);
uint dep = dx.deposit(0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2, 0.1 ether);
uint bought = dx.postBuyOrder(0xdd974D5C2e2928deA5F71b9825b8b646686BD200,0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2, 1051, 0.1 ether);
dx.claimBuyerFunds(0xdd974D5C2e2928deA5F71b9825b8b646686BD200,0xC02aaA39b223FE8D0A0e5C4F27eAD9083C756Cc2,user,1051);
uint kncbal = dx.balances(0xdd974D5C2e2928deA5F71b9825b8b646686BD200, user);
dx.withdraw(0xdd974D5C2e2928deA5F71b9825b8b646686BD200, kncbal);       

IERC20(0xdd974D5C2e2928deA5F71b9825b8b646686BD200).approve(0x49c4f9bc14884f6210F28342ceD592A633801a8b, kncbal);
uint256 ethBought = IUniswapV1Exchange(0x49c4f9bc14884f6210F28342ceD592A633801a8b).tokenToEthSwapInput(kncbal,
1,block.timestamp + 1 hours);
```

## Chal 3 - Too big to fail

We need to liquidate an underwater position before [this top-up](https://etherscan.io/tx/0x27964fe829e72020b66544df299f87ae6ee4202ff2394cdfe670d53754ca68fb) happened. This challenge was the most straightforward one, with just a two-line solve, liquidating the address doing the top-up:

```java
ITroveManager troveman = ITroveManager(0xA39739EF8b0231DbFA0DcdA07d7e29faAbCf4bb2);
troveman.liquidate(0x903d12bf2c57A29f32365917c706ce0e1a84Cce3);
```
It was the simplest challenge, but also the most mindblowing one! The fact that this 2 lines of code, executed at the right time, would net >2500 ETH is insane. 

## Chal 4 - First blood

We need to find "the transaction signature of the first attempt to snipe it, as well as the signature that actually made TRUMP trading possible". To start, look at the [first mint of TRUMP](https://solscan.io/tx/UFec7orzRrt17gDr6NpB1dNGwoKBSHyyEww94KJcZa4gFNYTBH2yWPVZDB4L5Gp4jsrPs9efwxWGdqGai4XKWtG) then start tracing what [txs happened around the time of the launch](https://solscan.io/account/9k8jWWqfmTTXrc8gmZqYVxSFJhygURvyVRztivJXFTYP). I saw a few things, like [this 4M TRUMP transfer](https://solscan.io/tx/RbX7T6LHzMSG8Xganz5bkmNPKYSF8jhTVTVStRJFte7WQHBAgHut2Ef5MmSyQJQjtZg9HmeG1t5QRDeyNZCwKKD) to [2Fe4...WSkv](https://solscan.io/account/2Fe47zbh8svDNGNehFy1NY8bsjQNtomvKFuq1jNgWSkv). Paging further back on this [2Fe4...WSkv](https://solscan.io/account/2Fe47zbh8svDNGNehFy1NY8bsjQNtomvKFuq1jNgWSkv) account I saw an [add-liquidity tx to Meteora](https://solscan.io/tx/5TLm2sua76vwR3c29MRhQYCrPgkdZAWKjaPd89wS9c51KTx9eQ6RNS7Bmq9BKfSHBP6kUUAdFJ2Yr6HEVuL2iZmr). I also saw that the pool itself was [seeded with TRUMP](https://solscan.io/tx/3QHPXjD3b7KzXHk51Gg4DDqBK5pr5yVkgxF4tN1DS84dv3X9AsPf7wUrU8ZCFrJeFo6YXTDkMcqfrv2axyuKANM2). I then inspected the earliest txs to the pool, and saw that there was a `togglePairStatus` call by [5unT...eTLi](https://solscan.io/account/5unTfT2kssBuNvHPY6LbJfJpLqEcdMxGYLWHwShaeTLi). This wasn't the correct one, but filtering by this account we find another `togglePairStatus` call which is the [correct one](https://solscan.io/tx/4SMUTho76nrPXxGNdDBNdBNbtbSC48oDDkivVKSdWUJR8KZGQwv1tEwJnHFXmpFDFkkLRupzzW28e6HHpv49afQt).

To find the first snipe, just order Meteora pool txs by oldest first, and look for the [first transfer after the first `togglePairStatus` call](https://solscan.io/tx/41h3CuLHamSdfsmgWC887eoyvrTiUcGjhLZpKMeqE9Rg9ZkP42C2gBr5PrQM9D25jRFwwQYPfBUJYCEUXC1qAxcv), which is the first attempted swap for TRUMP.

## Chal 5 - Smart Money

In this challenge we need to do some digging to find VCs and fundraises behind some addresses. I tried querying with Dune and some other analytics tools but didn't find these to be of too much help so I went on-chain. Looking at the tx history of [0xc29af06142138f893e3f1c1d11aa98c3313b8c1f](https://etherscan.io/txs?a=0xc29af06142138f893e3f1c1d11aa98c3313b8c1f&ps=100&p=3) I noticed there was a [transfer out to quilamir.eth](https://etherscan.io/tx/0x2f086a1861da38d78267067f61d9aa6fd43b13b01e87b29ac67abf79fc500f5a) and decided to investigate further. Searching `quilamir` gave me [their Github](https://github.com/Quilamir) and a [Sotwe link](https://www.sotwe.com/quilamir). I saw a retweet of Node Capital on their account, which tells us which VC is behind the address in question.

![node cap](/assets/img/sample/wintermute/node-cap.png)

From the [Node Capital website](https://node.capital/#portfolio), I looked through the portfolio and noticed there were corresponding blog posts for different fundraising rounds. Looking at the [earliest fundraising](https://etherscan.io/tokentxns?a=0xaf0970a06bd17ad57f63d633be7f7039eb69dc95&p=5), we can see that the date corresponds to around October 2021, and there's a [corresponding article](https://node.capital/blog/combine-conquer/index.html) for an investment in C3 protocol, which is the answer to which protocol was raising funds.

For the second half of the challenge, the company behind 0x4c2c0F0bB2631B02aC9299C59690914ee7A200B8 is bridge.xyz, as labelled by Etherscan. Then to figure out which protocol was raising funds from 0xe53ec250fDF41e52d22fEF1f76DeE92A9377AC8f, we can again look at the ENS names that [sent money to this address](https://etherscan.io/tokentxns?a=0xe53ec250fdf41e52d22fef1f76dee92a9377ac8f&ps=100&p=2). I found `coloso.eth`, `franalgaba.eth`, `maerg.eth`. Since the txs happened around the [end of 2024](https://etherscan.io/tx/0x597c1c64c5976e6d9fe91f53c95fc4e7d501ea8f8096745767e96deb2c5dc7b3), I looked at Fran Algaba's (from `franalgaba.eth`) twitter and saw that they were raising for Aligned Layer, which is the answer:

![fran](/assets/img/sample/wintermute/fran.png)

## Chal 6 - Cold start

For this challenge we need to access the Uniswap market on Robinhood [through the chain's delayed inbox](https://docs.arbitrum.io/how-arbitrum-works/deep-dives/transaction-lifecycle#bypassing-the-sequencer). The relay part is simulated in the code, so all we need to do is submit the UniV3 purchase tx to the delayed inbox. I ended up using the [retryable example from the dev docs](https://docs.robinhood.com/chain/cross-chain-messaging/). There's no need to wrap+bridge ETH as well because the ETH goes to the router which handles everything. The code to purchase CASHCAT through the delayed inbox is as follows:

```java
IUniswapV3Pool pool = IUniswapV3Pool(0xA70fc67C9F69da90B63a0e4C05D229954574E313);                

address weth = 0x0Bd7D308f8E1639FAb988df18A8011f41EAcAD73;        

IV3SwapRouter.ExactInputSingleParams memory params =
    IV3SwapRouter.ExactInputSingleParams({
        tokenIn: weth,
        tokenOut: 0x020bfC650A365f8BB26819deAAbF3E21291018b4,//cashcat
        fee: 10_000,
        recipient: user,
        amountIn: 9.95 ether,
        amountOutMinimum: 0, // PoC only
        sqrtPriceLimitX96: 0
    });

bytes memory swapData = abi.encodeCall(IV3SwapRouter.exactInputSingle, (params));

uint256 submissionCost = IInbox(INBOX).calculateRetryableSubmissionFee(swapData.length, block.basefee);

IInbox(INBOX).createRetryableTicket{
    value: 9.95 ether + submissionCost + 500_000 * 20_000_000// (amt in + feebudget)
}(
    0xCaf681a66D020601342297493863E78C959E5cb2,
    9.95 ether,          // l2CallValue: native ETH delivered to router
    submissionCost,
    user,         // excess fee refund
    user,         // call value refund
    500_000, //gaslimit
    20_000_000, //maxFeePerGas,
    swapData
);    
```

## Chal 7 - Firepit

This challenge asks us to [release TokenJar assets in exchange for 2000 UNI](https://github.com/Uniswap/protocol-fees). I've already attempted this on Ethereum mainnet so I had some idea about how the mechanics worked. What I didn't expect to see was that the chain in question had live, real money still sitting on the `Tokenjar`.

![magpie vibes](/assets/img/sample/wintermute/realmoneyfr.png)

This led me down a rabbit hole to see if I could grab this money for myself, knowing it was probably unlikely. Of course, it wasn't possible because there's no way to bridge the UNI needed by the releaser because the bridge itself was [neutered](https://etherscan.io/address/0x2978527d5D1372C32fEdC182FDE7559c0471d051#code) to disallow ERC20 bridging. Moreover, the actual atomic `release->bridge->burn` mechanism wasn't implemented and would [always revert with the current implementation of the bridge](https://github.com/okx/optimism/blob/40a442a7b0ffe3ff8d16156fbeba676e831c7c98/packages/contracts-bedrock/src/L2/L2StandardBridge.sol#L137). However, my rabbit hole didn't end there 😬 I got fully nerdsniped for _many_ hours by the alternative chains hosting Uniswap Tokenjars because there was so little competiton elsewhere, and what seemed like quite good profits to be made. I'm planning to writeup a full blog post just talking about this series of rabbit holes, but there's still a few days left while I try to wrap up this severely mis-scoped task I found myself in.

Anyway, for the actual challenge, I solved it by taking the easy route of flushing a bunch of V3 pools. For the sake of clarity, the protocol fees mechanism works like this:

1) Fees are enabled across UniV2/V3/V4 pools as decided on by governance
2) Fees flow to a central contract, the TokenJar. Fees flow either as pools are traded against (as in UniV2) or need to be flushed (as in UniV3/V4). 
3) Anyone can claim up to 20 tokens from the TokenJar, provided they burn a required amount of UNI tokens, usually 2000 or 4000 tokens. 

I figured the easiest way to solve the challenge would just be to flush a bunch of V3 pools and then sell the tokens to USDT. Flushing a lot of V3 pools is an easy way to solve the challenge because even if the pool is trading some garbage for a stablecoin, both fees get flushed to the Tokenjar, accumulating the stablecoins we want. I used 2 pages of the [Coingecko API](https://api.geckoterminal.com/api/v2/networks/x-layer/dexes/uniswap-v3-x-layer/pools?page=1) to populate pools to flush and then released the TokenJar. I selected Wrapped OKB, some stables and a bunch of Wrapped stocks and then sold those back off to UniV3 pools to get USDT. 

![our selected](/assets/img/sample/wintermute/pit.png)

You can find the final code solution [here](https://gist.github.com/toastedsteaksandwich/2c09c2dfe4b6c40e74f2e2f8ef6f4e2c#file-07solution-t-sol).

## Chal 8 - First move

This challenge seems quite straightforward on the surface: find a valid `claim` for an `attack(rootClaim, 0, claim)` call for a fault dispute game. In hindsight it's not super crazy, but I spent so many hours going down several rabbit holes, mainly because I found out I wasn't super well versed with how dispute games actually work and I was unfamiliar with Go when trying to read the op-challenger codebase. I'll try to walk through some of my thinking without brain-dumping my entire process. First, let's talk about some basics:

- What is a Fault Dispute Game? It's a type of dispute game where actors interact to verify the validity of a root claim. 
- What is a root claim? A root claim is _not_ a merkle root of a binary tree containing every state of an L2, rather it's the starting claim at the root of a dispute game tree. It's saying, _this_ is the L2 output root at a specified L2 block number.
- Why attack a root claim? In the case of the example, the root claim was `0xdeadbeef...deadbeef`, which is an obviously bogus output root. But in general, someone might want to attack a root claim ecause the claimed output root does not match the correctly derived L2 state.
- How do I attack the root claim? An attack is formed with a _claim_. This is saying "at an earlier point in the output root trace" (i.e. list of output roots), I think the output root was _this_ hash `0x123...`.  
- What happens after an attack? Actors bisect the output root trace to narrow the disagreement over L2 block outputs. If it bisects further past `SPLIT_DEPTH`, then they keep narrowing down the execution until they can check a single instruction.

My first problem was that I didn't answer these questions in this nice order. Initially, I spent time looking at an [actual game with the same root and attack](https://disputes.slopo.net/game/0x5551599a52b4c7fe09f578129879af00a433997f) and went through the [FaultDisputeGame code](https://etherscan.io/address/0x5551599a52b4c7fe09f578129879af00a433997f#code) only to find that there wasn't a whole lot of validation done on the `claim`. The only thing I could see was that we needed `uint8(_rootClaim.raw()[0])` to be INVALID or PANIC, but this meant that there were claims that could pass the submission checks for an attack, but the challenge is looking for _the_ valid claim as computed by `op-challenger`. Moreover, these checks are only triggered after we get tp depth `SPLIT_DEPTH + 1`, making this rabbit hole a dead end. 

I learnt that the claim I'm looking for is generated by the `op-challenger`, a challenger agent for dispute games. From here, I went down [several](https://github.com/search?q=repo%3Aethereum-optimism%2Foptimism+AttackTx&type=code) [rabbit holes](https://github.com/ethereum-optimism/optimism/blob/904a4093bd12da2b842e1aa4828eea237cb7e8a1/op-challenger/game/fault/responder/responder.go#L116) [many](https://github.com/ethereum-optimism/optimism/blob/develop/op-challenger/game/fault/agent.go#L285) [many](https://github.com/ethereum-optimism/optimism/blob/develop/op-challenger/game/fault/agent.go#L207) [layers](https://github.com/ethereum-optimism/optimism/blob/develop/op-challenger/game/fault/solver/game_solver.go#L98) [deep](https://github.com/ethereum-optimism/optimism/blob/develop/op-challenger/game/fault/solver/solver.go#L80) [to](https://github.com/ethereum-optimism/optimism/blob/develop/op-challenger/game/fault/solver/solver.go#L138) [find](https://github.com/search?q=repo%3Aethereum-optimism%2Foptimism+Get+trace&type=code) [how](https://github.com/ethereum-optimism/optimism/blob/904a4093bd12da2b842e1aa4828eea237cb7e8a1/op-challenger/game/fault/trace/access.go#L27) [AttackTxs](https://github.com/search?q=repo%3Aethereum-optimism%2Foptimism+Get+provider&type=code) [are](https://github.com/ethereum-optimism/optimism/blob/904a4093bd12da2b842e1aa4828eea237cb7e8a1/op-challenger/game/fault/trace/alphabet/provider.go#L66) [generated](https://github.com/ethereum-optimism/optimism/blob/develop/op-challenger/game/fault/solver/solver.go#L136) (❗), but this did not prove useful immediately. 

I went back to the drawing board a few times and tried to simplify what we needed: we're trying to find an earlier point in the output-root trace that we think is valid. From further poking around, I found that `TraceIndex` gives us the rightmost leaf for a given position in the game tree. It seemed like this is what we wanted, since the attack claim corresponds to the output at its rightmost covered leaf. And since we're at depth 1, we are looking at blocks, since it's only after SPLIT_DEPTH that we start bisecting into instructions. After some further looking around, I found that this [ClaimedBlockNumber](https://github.com/ethereum-optimism/optimism/blob/33fbe016b879d3c4a5187516cbab7668b4527627/op-challenger/game/fault/trace/outputs/provider.go#L58) function might be useful since it fetches the TraceIndex from a position and hands us the block number. We can work this out [manually](https://github.com/ethereum-optimism/optimism/blob/33fbe016b879d3c4a5187516cbab7668b4527627/op-challenger/game/fault/trace/outputs/provider.go#L64):

```
o.prestateBlock = startingBlockNumber():= 52959235
o.poststateBlock = l2BlockNumber():= 1787098259
o.gameDepth = splitDepth():= 30
```

and with some [python code](https://gist.github.com/toastedsteaksandwich/2c09c2dfe4b6c40e74f2e2f8ef6f4e2c#file-fetch_trace-py) to calculate the `traceIndex` from position 2 we have `traceIndex + prestateBlock + 1`:

```
536870911 + 52959235 + 1 = 589830147
```

However, it turns out we don't need this `claimedBlockNumber` but rather the `HonestBlockNumber`, since the `claimedBlockNumber` might be after the [safe head](https://github.com/ethereum-optimism/optimism/blob/33fbe016b879d3c4a5187516cbab7668b4527627/op-challenger/game/fault/trace/outputs/provider.go#L83) (note that a safe head is a block marking how far L2 can be reconstructed from the Ethereum data so far, the game's `l1head`). Where do we get this? There is a rollup RPC method `optimism_safeHeadAtL1Block`, but it turns out this method is not supported anywhere I could find. What now? Luckily, we can use block explorers to see what batches have been posted. As far as I understood, the last block in a valid batch indicates the safe head at the time. So we need to find the batch around the time of the l1head given to us from the challenge:

```
$ cast block   0xd74f339891bc1c4af93bf4bb55c03fc3feb62da272a1f7aa3eabb9e2410126f4 --rpc-url https://eth.drpc.org --field number

> 25785478
```
Then look at [block 25785478](https://etherscan.io/block/25785478) on the explorer to see that it's on Aug-19-2026 12:10:47 AM +UTC. With that, start looking through batches https://explorer.inkonchain.com/batches like https://explorer.inkonchain.com/batches/50975 and start tweaking the batch till it corresponds to the L1 submission timestamp to ETH block 25785478. Eventually, we find [batch 46834](https://explorer.inkonchain.com/batches/46834) and the latest block in this batch is 53599386. Now to find the output root at this block, we can query the `optimism_outputAtBlock` method, and luckily it seems to be supported by Quicknode. The [demo in the docs is for Ink sepolia](https://www.quicknode.com/docs/optimism/optimism_outputAtBlock), but just editing the endpoint to mainnet works 😂

```bash
curl -X POST "https://docs-demo.ink-mainnet.quiknode.pro/" \
  -H "Content-Type: application/json" \
  -d '{
  "jsonrpc": "2.0",
  "method": "optimism_outputAtBlock",
  "params": [
    "0x331dc9a" //cast to-hex 53599386
  ],
  "id": 1
}'
...
0x82c941153a9de14c4533b301799ee33206b6a475d7c4fdbe7cd2f1c9d7271b6f
```

And that's the claim. Do the same for Optimism, with the latest L2 block available at the same L1 cutoff being 155749670 in [batch 490771](https://explorer.optimism.io/batches/490771):

```bash
curl -X POST "https://docs-demo.optimism.quiknode.pro/" \
  -H "Content-Type: application/json" \
  -d '{
  "jsonrpc": "2.0",
  "method": "optimism_outputAtBlock",
  "params": [
    "0x9488d26" // cast to-hex 155749670
  ],
  "id": 1
}'
...
0x192f163548d61d555a282e1ffcec8ec7b1e4cf9deced7e910b87292f0aeab5f1
```

Which gives us the final solve :)

![Finally!](/assets/img/sample/wintermute/yipee.png)


## Final remarks

Overall I had a lot of fun! I really enjoy MEV/MEV-adjacent CTFs and I'm glad I could do this one since I was sad to miss last year's competition. I haven't been able to commit full weekends to CTFs like I could in earlier years, so not having a leaderboard and a time limit worked in my favour this time. I'm still working on the Tokenjar searching situation on live networks, so the blog post on that will hopefully come soon since I found some interesting artifacts to share. 

I'd love to do more of this type of work irl, so please hit me up! I'm currently flip-flopping between bug bounty and MEV assisted with AI, and I've never learnt searching in a more formal way and I've become quite keen on it.

---
title: 🍅🍞🐟 Searchers of Nottingham Writeup 
author: Ashiq Amien
date: 2024-07-02 13:52:00 +0200
categories: [CTF, MEV]
tags: ctf
math: true
---

# Intro

[Searchers of Nottingham](https://nottingham.dragonfly.xyz/) was a MEV-style CTF game that ran for 2 weeks over 3 seasons. The game provided a market that allowed players to trade against each other with the aim to have the most of any one good according to the winning criteria. The game aimed to emulate what MEV searchers might need to do in a real market, by outpacing or exploiting other traders through various strategies. This blog post is a walkthrough of my attempt at the game over the 3 seasons :)   

![irl pic of searchers of nottingham](/assets/img/sample/nottingham-blog-post/intro.png)

## Outline

The provided market was an n-dimensional constant product AMM. Compared to Uniswap V2, all goods (fish 🐟, tomatoes 🍅 and bread 🍞) were traded against a shared reserve, gold. The market was seeded with liquidity, and each player gets 1 unit of each good and 1 wei of gold every round. The game is played until any one player has 64 units of a good, or whoever has the most of a single good at the end of 32 rounds. 

Players have 2 ways to trade- either by submitting a bundle of trades, or by participating in a blind auction to become the block builder, who can access the market directly and settles all other players' bundles atomically. Participation is done through a smart contract, called an agent, that performs these trades/bidding for a full game. Games always have 4 players, 3 non-gold assets, and gold, which doesn't count towards the winning criteria. 

## Season 1 - Sandwiching over goldmaxxing

To start, we're given a few sample agents to play around with. For example, the `SimpleBuyer` agent buys a single good for the duration of the game and does not bid anything, while the `GreedyFrontRunner` tries to buy whichever good it has the most of and bids to try and get builder rights to swap first. I submitted the `GreedyFrontRunner` agent just to see how things will pan out in the game:

![GreedyFrontRunner](/assets/img/sample/nottingham-blog-post/1.png)

Let's get some intution of how this agent works by looking at the graph and a few rounds of the match. At some point, there's a spike of the number of tomatoes bought. This is kept consistent afterwards until the 32 rounds are done. At round 22, my blind bid wins and I buy all the tomatoes 🍅 I can by selling my other goods, and I do this before everyone else to get the best price (naively). Intuitively, it seems like being the block builder is powerful and worth bidding a large amount of gold. More importantaly, gold is *not* important for the winning criteria, so it makes sense to get rid of it anyway. 

### The first strat 

After getting some intuition, some possible first strategies come to mind:

- Get much gold as possible, and once we have enough, bid a lot to buy 64 units of something. (called goldmaxxing)
- Get a lot of gold, and bid+frontrun when someone else is close to winning.
- Consolidate into one good, and if I can get more of a single good in the next round, swap it. (e.g. I have 30 tomatoes, but if I swap I can get 32 bread. Remember, we need the most or 64 of any one good).
- Other agents aren't bidding at all. Bid at least 1 wei. If they start bidding 1 wei, start bidding 2. If we're builder, settle user bundles in a way that gives us the most goods, not just first or last.

I landed on using the first idea:

```javascript
function createBundle(uint8 /* builderIdx */)
        public virtual override returns (PlayerBundle memory bundle)
    {
        bundle.swaps = new SwapSell[](MAX_SWAPS_PER_BUNDLE;
        for (uint8 assetIdx; assetIdx < ASSET_COUNT; ++assetIdx) {                    
                uint256 bal = GAME.balanceOf(PLAYER_IDX, assetIdx);
                bundle.swaps[assetIdx] = SwapSell({
                    fromAssetIdx: assetIdx,
                    toAssetIdx: 0,
                    fromAmount: bal 
                });    
        }            
        uint256 maxGoodsAmount;
        uint8 targetAsset = 0;                
        for (uint8 assetIdx = 1; assetIdx < ASSET_COUNT; ++assetIdx) { 
            uint256 out = GAME.quoteSell(0, assetIdx, GAME.balanceOf(PLAYER_IDX, 0));                
            if(out > 65e18){ 
                targetAsset = assetIdx;
                bundle.swaps[ASSET_COUNT] = SwapSell({
                        fromAssetIdx: 0,
                        toAssetIdx: targetAsset,
                        fromAmount: GAME.balanceOf(PLAYER_IDX, 0)
                    });   
                break;
            }
        }            
    }

    function buildBlock(PlayerBundle[] calldata bundles) public virtual override returns (uint256 goldBid)            
    { 
        //settle ours first
        for (uint8 playerIdx = 0; playerIdx < bundles.length; ++playerIdx) {
            if (playerIdx == PLAYER_IDX) {
                GAME.settleBundle(playerIdx, bundles[playerIdx]);
            }            
        }          
        goldBid += GAME.sell(
                    _getMaxGood(),
                    GOLD_IDX,
                    GAME.balanceOf(PLAYER_IDX, GOLD_IDX) * 1/100
                );                   
        for (uint8 playerIdx = 0; playerIdx < bundles.length; ++playerIdx) {
            if (playerIdx == PLAYER_IDX) {
                // Skip our bundle.
                continue;
            }
            GAME.settleBundle(playerIdx, bundles[playerIdx]);
        }                                
    }   
```
This had some success- we can see the graph hockey stick up at the end, where my agent bid aggresively:

![hockey stick up](/assets/img/sample/nottingham-blog-post/2.png)

But it also had some misses, where the agent sold it's goods when it didn't hit the 64 mark after frontrunning:

![front run again](/assets/img/sample/nottingham-blog-post/3.png)

Why did this happen if we had a quote? It's important to remember that the bundle we craft in `createBundle` vs the actions in `buildBlock` are under very different contexts. Any assumptions about the market can be skewed in bundle from `createBundle` if the builder places other transactions around our bundle. This is what happened in the above picture, where our `GAME.quoteSell(...)` call returned >65e18 when being crafted, however, we were frontrun when our blind bid wasn't high enough. If we're the builder, we have full context and control of the market for that block, besides needing to execute the user bundles. 

### The second strat

This naturally leads into the next strategy, which came about on the last day of the season from another user. User aaa realized that being the builder allows us to sandwich another user's bundle:

![the first sandwich](/assets/img/sample/nottingham-blog-post/4.png)

By inspecting the bundle contents and seeing what the user wants. By being the builder we can frontrun the trade, push the price of the target goods up (it's a constant product AMM, remember) and then backrun the trade for a reliable profit. In this picture, we can see that user aaa made a *free* profit of about 0.1 tomato and 0.1 fish. I say *free*, but of course you need to win the blind auction for builder rights. Compared to Uniswap V2, trades have no slippage tolerance, so you can sandwich another users trade with your entire balance.  
 
When developing my second strategy, I aimed to be aggresive with my bidding strategy to try and always get builder rights so I could reliably profit from my sandwiches. I also tried to be aware of what other players were doing, e.g.:

- merklejerk always bids half of their balance, so bid more than that
- gyattmaster69 does not bid, so any bid beats that
- student does not bid unless they're about to purchase 64 units of goods, and bids the remaining amount
- gmluqa and guzus sells 5% of 2 non target goods and bids that for gold
- aaa bids 60% of the sum total of the last trades into gold. The amount of goods traded into gold is 70% of the balance

So my strategy was as follows:

- write a sandwich alg in `buildBlock`
- bid more than half of merklrjerk's balance, and more than aaa's 60% of gold from 70% of their balance of goods
- if we have enough gold, buy up to 64 units of a good and bid the balance of our portfolio

```javascript
...
// sandwich everyone
for (uint8 playerIdx = 0; playerIdx < bundles.length; ++playerIdx) {        
    if (playerIdx == PLAYER_IDX) {
        // Skip our bundle.
        continue;
    }
    //get the most important asset in the bundle to sandwich
    uint8 targetSandwichAsset;
    uint256 targetToAmount;
    for (uint256 curTx = 0; curTx < bundles[playerIdx].swaps.length; curTx++) {
        SwapSell memory currentTx = bundles[playerIdx].swaps[curTx];
        if(currentTx.toAssetIdx != 0){
            //check which swap will give us the most out
            if(GAME.quoteSell(currentTx.fromAssetIdx, currentTx.toAssetIdx, currentTx.fromAmount) > targetToAmount){
                targetToAmount = GAME.quoteSell(currentTx.fromAssetIdx, currentTx.toAssetIdx, currentTx.fromAmount);
                targetSandwichAsset = currentTx.toAssetIdx;
            }
        }                
    }
    //now, buy targetSandwichAsset with everything we have since we're only in gold
    for (uint8 assetIdx; assetIdx < ASSET_COUNT; ++assetIdx) {                                        
            GAME.sell(
            assetIdx,
            targetSandwichAsset,
            GAME.balanceOf(PLAYER_IDX, assetIdx)
            );  
    }
    GAME.settleBundle(playerIdx, bundles[playerIdx]);    
    //now, sell everything back for gold                                                   
    GAME.sell(
        targetSandwichAsset,
        GOLD_IDX,
        GAME.balanceOf(PLAYER_IDX, targetSandwichAsset)
    );      
}
    ...
```

Unfortunately, this wasn't enough to win. I bumped up to 71% and 81% on aaa's bids in an attempt to outpace their anticipated increase. What I didn't notice was that on the very last round before submissions closed, aaa already increased their bids and I didn't recalculate it. I'm fairly sure that with a higher increase my agent would have won, but at the same time, having too much gold to bid might have increased the number of rounds it'd take to buy 64 units of a good, so I can't be too sure. 

## Season 2 - Salmonella 

At the beginning of season 2 I was a bit unsure about how the game would progress, what else could there be besides goldmaxxing and sandwiching? It wasn't long until I noticed some interesting my strategy had a major flaw:

![the sandwich is a mess](/assets/img/sample/nottingham-blog-post/5.png)

If you look closely, I actually lose gold on my sandwich trade, something that should never happen. As I was figuring out how to fix it, another meta was coming up:

![salmonella sandwich](/assets/img/sample/nottingham-blog-post/6.png)

I noticed that several bids were failing whenever `plotchy` was involved. Somehow, this user caused bids to fail. This means that there was a revert in the `buildBlock` function, either from an underflow or for some other reason, and `plotchy` was able to force it. And even while I was investigating this, a third meta came up:

![anti club sandwich sandwich club](/assets/img/sample/nottingham-blog-post/8.png)

Here, `legion2002` was an active sandwicher, but did no trades when not granted builder rights. Of course, one great way to fight off someone frontrunning and backrunning your trades is to have none at all. This strategy allows a user to retain all of their goods until they're granted builder rights at a later round. Since they retain more goods, their bid can be higher and will gradually get higher since they simulate larger and larger trades for themself when deciding how much to bid.

At this point, I realized that there is no one set-and-forget agent. This game, and I guess actual on-chain MEV, is about adapting to the current meta and always trying to one-up your opponenents. Users were getting exploited, sandwichers were no longer making money from certain users, none of which was present in season 1.

### The third strat

My third strategy was fixing up some of the mistakes from the previous agent (losing money on sandwiches, getting reverts, etc) and just trying to play whatever meta comes up. While no-trade bundles were popular at the time, I noticed that I was still coming out with the [highest gold balance at the end of the first few rounds](https://nottingham.dragonfly.xyz/match?tournament=0e07f422-e0b0-4c79-85b9-90654738d991&season=2&bracket=3&match=c13dd8fe-bd7e-4526-96a5-8846d1020e19&idx=12) when selling for gold, even if I was being sandwiched. This is because the sandwicher bids a lot to get builder rights, and no-op trades don't affect the market. Lastly, your rank depends on your position at the end of the game, so coming out second or third place consistently is better than coming first every now and then. 

At the end, I had the following:

- Improved sandwiching that tries to follow all directions of the trade e.g. if some goes to gold, follow that as well instead of taking the highest direction trade from before.
- Sell all goods for gold in the first couple rounds since goods go for highest in the first rounds. This is to have bidding power later on.
- Follow the rules of the `simpleBuyer` agent after the rounds of selling for gold are done
- Cap the bid at the gold balance to prevent underflows
- Continue to try and buy up to 64 units and bid the remainder.

Unfortunately, this wasn't enough either, and I landed up in 4th place. From what I can tell, the winner `legion2002` used aggressive bidding to sandwich when block building, as well as submitting empty bundles when not block building, to win first place. Being builder allowed them to exploit users for profit, while retaining the profits in no-op trades.

## Season 3 - Leaky liquidity

Season 3 introduced a change where the market was seeded with a higher liquidity, but leaked assets evenly each round. To start season 3, there were a few optimizations I wanted to make to my agent regardless of the current meta:

- batch sandwiches together, i.e. if 2 bundles (have a majority) trade into one asset, settle both bundles before backrunning
- hotswap the max goods I have for any other type of good when I'm the builder (e.g. 22 fish into 24 tomatoes)
- try to poison user bundles during the simulation stage? (the simulation stage is when user block building is simulated to get the bids for the blind auction)
- figure out if we can construct our bundle in a way that minimizes the amount of profit a sandwicher can take (e.g. 75% into target good, 25% into gold)

The first two items were straightforwad to implement. I then tried to do the fourth point, and spent sometime on-paper to figure out how to construct such a bundle. After a long time, I realized I rabbitholed into nothing- our bundles do *not* have access to the current market. Bundles are submitted in this form:

```javascript
/// A structure representing a sell order.
struct SwapSell {
    // Index of the asset to sell.
    uint8 fromAssetIdx;
    // Index of the asset to buy.
    uint8 toAssetIdx;
    // Amount of the asset to sell.
    uint256 fromAmount;
}
```
This means that we can't pass in some function call into our bundle to figure out how much to buy and sell based on the current reserves, it needs to be some uint. I eventually just dropped this since I was running out of time. I played around with bundle poisoning and submitted bundles with an invalid `toAssetIdx`, as follows:

```javascript
bundle.swaps[i] = SwapSell({
    fromAssetIdx: assetIdx,
    toAssetIdx: type(uint8).max,
    fromAmount: 1
});
```
and sure enough, I forced some reverts:

![leftover salmonella](/assets/img/sample/nottingham-blog-post/9.png)

I didn't want to leave this ""alpha"" on the table so I switched back to another agent and patched up my own current agent to make sure that the `toAssetIdx` is always in a valid range. At a later stage, I found my own agent being poisoned (thanks weeb_mcgee) because I didn't validate the incoming `fromAssetIdx` either 😂 

### The final strat

Towards the end of the game it was clear that were 2 main approaches to take- take the riskier approach to have more gold and aim to get 64 units quickly, or slowly grow your holdings up to 64. Since there were a lot of players going with the first approach, I tried to go in the middle of maintaining some target asset and if I have enough, buy up to 64. I wanted to consistently be ranked higher and hopefully win games occasionally, instead of win sometimes and have no points otherwise. In the end, I had an agent that did the following:

- Submit bundles that sell 85% of everything for some target asset, and 15% for gold
- Sandwich other users if I become the block builder
    - Batch bundles that have the same target asset when sandwiching
- When blockbuilding, try buy up to 64 of *any* good with the entire balance
    - if I can, run through player balances to make sure I have higher than them (66 vs 64 means the player with 66 wins regardless of player order)
    - if I win, sell everything else for gold and bid the balance
- If I don't have 64 of a single good, sell 10% of my max good balance for gold, and bid between 1-15% of my gold balance if the game is less than 15 rounds.
    - If the game is passed 15 rounds, sell only 5% of the max good balance but bid 40-60% of my gold balance. This is to bid more gold while retaining more of our max good while the game approaches the end.

Unfortunately, this wasn't enough and again landed me in 4th place. My bidding strategy accounted for my whole balance and not just sandwich profits as it did in previous versions of my agent, which lead me to leak of a lot of gold when trying to sandwich empty bundles. User `cvpfus` who came first used empty bundles, and an aggresive bidding strategy when coming close to a win. ggwp!! 

![final results](/assets/img/sample/nottingham-blog-post/10.png)

## Conclusion

Overall this was a super fun CTF/game, and it's nothing like I've played before. It was definitely challenging in a different way compared to other CTFs and you're forced to adapt. I struggled a bit with testing locally vs testing in the arena since you can't write other user's agents during the season (unless you're a cracked anon) and there were still some things I didn't get to because of time. Either way, I had a lot of fun and learnt a lot :)


> I have to give a special mention about the design and security of the game- I spent a good amount of time looking around at the contracts to see where things might go wrong in the simulation, blind auction, etc. and everything was really well done. Kudos [merklejerk](https://twitter.com/merklejerk) and to the peeps at [Dragonfly](https://twitter.com/dragonfly_xyz)! `>|<`

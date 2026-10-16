---
title: 🍯 Scanning 28,758 Uniswap pools and minting synthetic UNI tokens to try and turn a profit of at least $420.69
author: Ashiq Amien
date: 2026-10-15 15:15:15 +0200
categories: [Research, MEV]
tags: research
---

![scale](/assets/img/sample/tokenjar/scale.JPG)

## Intro

I thought I found easy money that everyone else had missed. A few failed attempts later, I just wanted my own money back.

A few weeks ago I finished up the [Wintermute Alpha challenges](https://ashiq.co.za/posts/Wintermute-Alpha-Solves/) and noticed they left some actual alpha in challenge 7, showing that the [XLayer TokenJar](https://www.oklink.com/x-layer/evm/address/0x8Dd8B6D56e4a4A158EDbBfE7f2f703B8FFC1a754) had real money just waiting to be claimed. I decided to look around at the other `TokenJar` implementations on other networks and saw that there was some easy money there too! At least, that's what I thought initially, only to fall into a very deep rabbit hole and stumble several times before landing an actual, profitable `TokenJar` capture.

**TL;DR**

- I spotted an opportunity to make some profit off the [Uniswap Protocol Fees](https://github.com/Uniswap/protocol-fees) mechanism.
- I missed the first attempt, and a few more, only to find that competitors were likely monitoring who planned to capture the profit.
- I attempted to capture the same opportunity on the Optimism chain, only to miss the jar and find no practical way to exit immediately other than waiting out the 7-day challenge period.
- I decided to hedge my position and after more preparation, and some drama, finally claimed some profit.
- The strategy was profitable, but relying heavily on AI made execution much less reliable than I expected


## Background

Uniswap released a system to ["collect fees from any revenue source on any blockchain and convert them efficiently"](https://github.com/Uniswap/protocol-fees). The mechanism is fairly simple:


- Fees from Uniswap v2/v3/v4 accumulate and can be flushed into a single contract called the `TokenJar`.

- There is a releaser mechanism that anyone can trigger to release up to 20 tokens of their choice from the jar, provided the searcher pays a specified amount of UNI tokens, usually 2000 or 4000 tokens.

- The UNI is then burnt by the releaser in exchange for the tokens, allowing for efficient conversion.


This looks something like the following:

![jar mech](/assets/img/sample/tokenjar/mech.png)

The obvious initial strategy looks like this:

```
1) What tokens does the jar hold right now?

2) What is the sum of the top 20 balances worth?

3) What does the amount of UNI required by the releaser cost?

4) If it's profitable, buy or flashloan the UNI, `release()` the top 20 balances and liquidate all the tokens and net the difference as profit.
```

An important point is that the current jar token balances are a lot less important than the **fees** that need to be flushed from Uniswap v2/v3/v4 pools. Fees from pools (other than v2 pools) don't automatically flow into the jar and each version of Uniswap pools have different mechanisms to flush fees into the jar to be collected. When I was searching on mainnet I saw `jaredfromsubway` had already incorporated fee flushing+jar claiming as part of their MEV sandwiching so I decided to move elsewhere since I didn't think it was worth the time to compete. Nevertheless, from my recent adventures to chase a jar claim, I found these non-obvious hurdles that are worth mentioning:

- You need to evaluate the value of all tokens, so you need a way to enumerate all tokens, usually with some 3rd party API. Some token valuations are non-obvious, like Uni V2 LP tokens, or otherwise obscure tokens that aren't scam coins.

- You need a deeply comprehensive scan of **all** pools on the chain, or your jar valuation will not be accurate. If high valued pools are missed, the maximum value will decrease, leading to less profit and potentially leading to missing the jar if your profit target isn't met. Moreover, tokens should also be considered for their liquid value after slippage since that's what lands up in your wallet. It seems obvious, but if you overvalue the jar you'll lose money (perhaps due to stale prices), and if it's undervalued (as described above) you could be too late to capture the jar.

- Buying UNI exposes you to market fluctuations. Buying it too early exposes your position to more market fluctuations, while buying it too late may cause you to miss the opportunity to claim. Market direction also plays a big role in the window of opportunity to capture the jar.

## Tripping the wire

After looking at the Wintermute TokenJar challenge, I got Codex to start evaluating jars on other networks. Immediately I spotted that the jar on BNB was worth more than the current UNI cost by several thousand dollars (!) and decided I needed to act urgently to claim the jar. The first hurdle became immediately clear: the [resource](https://bscscan.com/address/0xa59FfbB55D91Fc32b44A06F0b9cc6036a4afbcE2#code#F14#L13) required was [`nttSyntheticUni`](https://bscscan.com/address/0x06e8bdE95BE4ce5cB1134BD47aD18a79fFB35822), a [Wormhole](https://Wormhole.com/)-wrapped token representing UNI from Ethereum mainnet. This immediately changed the dynamics of the capture in ways I didn't fully realize at the time, with the most obvious consequence being that a flash-swap capture was not possible. Liquidity for this token existed on Ethereum, but it then needed to be bridged across to BNB, so there were just a handful of tokens that could actually capture the jar, none of which was for sale. Since the profit was sufficient to justify taking on the risk of being the required UNI, I decided I'd buy the UNI and bridge it across while Codex was preparing the contract to capture the jar and liquidate the proceeds. However, even bridging was not that simple. The [Wormhole bridge UI](https://portalbridge.com/) did not support this tx, so I needed a script to interact directly with the [`NttManager`](https://etherscan.io/address/0x6569925Aac77D6B8Bb085F31F9828ff80D5a0c44) to wrap my UNI, as follows:

```bash
# Lock the UNI and send it to BNB.
# Wormhole's BNB chain ID is 4, not the EVM chain ID 56.
cast send "$ETH_NTT_MANAGER" \
  'transfer(uint256,uint16,bytes32)' \
  "$AMOUNT_RAW" 4 "$RECIPIENT_BYTES32" \
  --value 0 \
  --rpc-url "$ETH_RPC" "${SIGNER_ARGS[@]}"

# After fetching and verifying the Guardian-signed VAA,
# redeem it on BNB to mint 1 NUNI to the recipient.
cast send "$BNB_TRANSCEIVER" \
  'receiveMessage(bytes)' "$VERIFIED_VAA" \
  --rpc-url "$BNB_RPC" --legacy "${SIGNER_ARGS[@]}"
```

Since everything in this entire operation was vibecoded (more on this later), I bridged just 1 UNI to make sure things worked. Once verified, I initiated the bridge of the remaining UNI and [7 seconds later, a competitor came in and swept the jar](https://bscscan.com/tx/0x71f95ab780277bb8e2fc0b6e0de7fbc42710a3199c574719c3f3a97ab479a5a6):

![first miss](/assets/img/sample/tokenjar/first-miss.png)

I didn't think too much of it other than the competitor deciding to capture it because they may have just hit their own profit threshold since the jar was quite profitable. Unfortunately, I missed several consecutive profitable jars because of operational failures, inadequate jar coverage or an overly high profit threshold (w.r.t. having incomplete coverage). Several missed attempts later that it seemed like competitors were actively reacting to in-flight nUNI (synthetic UNI) bridges, especially if the in-flight nUNI amount and the recipient's balance add up to at least the required amount to capture the jar.

Of course, at this point I realized that searchers were actively monitoring the central [NttManager](https://etherscan.io/address/0x6569925Aac77D6B8Bb085F31F9828ff80D5a0c44) contract and that every time I bridged into it, I was tripping the wire to let their bots execute the claim if it was profitable for them. My first instinct was to split future bridges into smaller amounts, but it seemed overly convoluted and not worth the risk. Moreover, the UNI price was pumping and competitors were taking on thinner margins just to protect their upside from the pump. I decided I'd do the same, so I bridged my nUNI back to UNI and sold all my UNI at market until things subsided a bit.

## Optimistic protocol fees

After missing the jar a few times I was determined to get a profitable capture on *some* chain. I scanned all of the chains listed under the [protocol fees README](https://github.com/Uniswap/protocol-fees) and the chain that stood out was Optimism since it wasn't dead but wasn't overly competitive either like base or mainnet. I was largely familiar with what needed to be done, and since I wasn't greedy but rather just wanted some *non-zero* profit, I figured it should be quick and easy. I set up my jar monitoring and evaluation scripts, deployed and tested my smart contract to release the jar and enforce some non-zero profit margin on execution. I waited several days for the jar to come to the desired claim value and iterated over the system to ensure that the contract execution was sound, jar asset coverage was up to scratch, etc. However, when the jar valuation came close to the UNI replacement cost, there was a quick price dip and I got sniped **again**:

![Desktop View](/assets/img/sample/tokenjar/iota.png)

What went wrong? Despite asking Codex to learn from every missed snipe, it turns out there were several things that still went wrong:

- The first issue was that our jar coverage was not exhaustive, despite my several iterations of prompting to ensure accurate coverage with a high confidence level. This was unique to Optimism in that there were two v3 pools, WETH/WBTC and WETH/USDT, that were carried over during the [final regenesis](https://docs.optimism.io/op-mainnet/pre-bedrock-history/regenesis-history), leaving their original `PoolCreated` events absent from the post-regenesis history. This meant the jar valuation was off by ~0.26 WETH, so the competitor was able to hit their claim threshold much faster. An additional 101 pools were found after a postmortem, bringing the total number of scanned pools to 28,758. 

- The second issue was that the UNI price was declining sharply and my free-tier UNI pricing API was using a stale price. Had the system used a more accurate/stable price feed, the jar valuation would have been more attractive vs the UNI cost and potentially within the claim threshold.

- The third issue was that I set a very small profit threshold of $42.69, but this was still insufficient for a trigger since the max profit we hit was $32.12. I assumed $42.69 would be small enough to undercut the competitor entirely, but this was a useless barrier given the first 2 issues of accurate `jar<->replacement cost` valuations.

At this point I was pretty jaded since this project was meant to be a quick `for-fun-and-profit` situation I could use for a short blog post and use in job applications (I'm currently open to this kind of work btw!) but it had already taken a lot of time and exposed me to risk through the UNI spot position. I figured I could retry with some better planning, but it turned out I had bigger problems on my hand.

### A challenging 7-day window

Due to market volatility, I profited from the UNI upside while missing jar claims in previous attempts on BNB, Polygon etc, but the UNI price was now dropping close to my entry for the UNI I bought for this attempt. I'm well aware of how fast markets can change direction, so I figured I'd sell the tokens while I gathered myself before the next attempt. Unfortunately, the liquidity on the chain was insufficient, so selling would cause too much slippage and I wouldn't be able to recover my capital. I looked at a few CEXes and none seemed to support UNI as a deposit token despite UNI trading there. Then, I looked around at a few bridges to bridge back to mainnet and then sell it there, but the bridges either couldn't handle the size or didn't support the token at all. In fact, I almost got phished for the entire amount! I reported this to SEAL afterwards just fyi:

![Desktop View](/assets/img/sample/tokenjar/phish.png)

Now the last option would be to bridge back to mainnet with the canonical bridge. However, compared to the Wormhole-wrapped synthetic UNI used for BNB, Polygon and other chains, this was optimistic-bridged UNI meaning it was subject to Optimism's 7-day challenge period before it reached mainnet. This meant that even if I wanted to bridge my tokens back and take a short break, I couldn't! I'd be stuck in the insanely long challenge period while exposed to a spot UNI position I was increasingly uncomfortable with. I had a hunch that this price drop was going to be severe (partly because of the insane run-up that happened) and as a last option to save myself from a larger loss, I hedged my position on Hyperliquid with the UNI perp and went 1x short to go nearly delta neutral:

![Desktop View](/assets/img/sample/tokenjar/open-short.png)

I wasn't super happy with this situation and wished I had tested a round trip of bridging UNI to and from Optimism like I did with the synthetic UNI tokens, but I was there and needed to find a way out without taking a large hit through slippage. I could bridge back and wait seven days with the hedge open, or stay and try to claim the jar. If UNI kept falling, the short would be profitble and it'd bring the next claim sooner.

## The average-sized short

With my position hedged, I brushed off my feelings of fatigue and decided to setup again, trying to be aware of all possible pitfalls I had become aware of in the last few days/weeks. Firstly, I wanted to know who my competition was. The last claim was done by some entity called [`0xiota`](https://optimistic.etherscan.io/address/0x3f11753c51A8aCC79B78b729D477948b4b880da1) and interestingly, they used a contract that was [open sourced](https://optimistic.etherscan.io/address/0xD654DA253C815027154375aBdf3BaA97D0B46CFb#code) for the claim:

![jar mech](/assets/img/sample/tokenjar/UKR.png)

This was interesting because all other claims I'd seen had been executed through a private closed-source contract, while this had several [permissionless claim functions](https://optimistic.etherscan.io/address/0xD654DA253C815027154375aBdf3BaA97D0B46CFb#code#F1#L138) for anyone to use. I also thought it was interesting that it had Ukrainian in-line comments, can't say I've seen that before despite reviewing contracts for several years now. In any case, I thought this was a good backup option for our operations, so I got Codex to prepare a script to generate calldata for this contract as an emergency option if, for any reason, our setup didn't work, as well as a local script I could force execute. This made sense for me since `0xiota` had already used this contract for several claims, and I couldn't see any security issues with it. The only downside to using this contract was that it leaves the recipient to liquidate all the assets manually after the claim, which is part of the reason I left it as a secondary option.

The second hardening process I added was to enumerate all possible competitors on the chain. This is easier with chains that use synthetic UNI since there is no other purpose for them, so you could just list all holders and see who your competition is. On Optimism, the UNI used is the main variant that was created in the genesis block so there are many more holders and the UNI is actually traded as normal, even if the liquidity is low. That being said, I was still able to list only 3 other holders above the threshold 2000 UNI value:

![Desktop View](/assets/img/sample/tokenjar/holders.png)

Looking through each holder's tx history, I found that two were EOAs that had been dormant for several years, and the main holder was a Uniswap contract. I was confident that I was the only person on the chain interested in the `TokenJar` at the time, even if there wasn't definitive proof (i.e. UNI split over several accounts can easily be batched at claim time). To maintain this position, I added monitoring for incoming UNI bridges, with alerts sent to my Telegram:

```javascript
for log in response.json()["result"]:
    # Indexed topic 3 is the Ethereum sender.
    sender = "0x" + log["topics"][3][-40:]
    # Event data contains:
    #   word 0: Optimism recipient
    #   word 1: amount
    raw = bytes.fromhex(log["data"][2:])
    recipient = "0x" + raw[12:32].hex()
    amount_wei = int.from_bytes(raw[32:64], "big")
    amount_uni = amount_wei / 10**18
    tx_hash = log["transactionHash"]
    send_Telegram(
        "🚨 UNI bridge into Optimism detected\n\n"
        f"Amount: {amount_uni:,.6f} UNI\n"
        f"Ethereum sender: {sender}\n"
        f"Optimism recipient: {recipient}\n"
        "Status: L1 initiated; L2 finalization not yet observed\n"
        f"Transaction: https://etherscan.io/tx/{tx_hash}"
    )
```

This is to alert me that something is happening, but is otherwise largely cosmetic. The actual in-flight UNI monitor is used as a trigger in case there's an EOA whose UNI balance is about to be above the threshold amount, which means it's more than likely that that person is a competitor for the jar.

### Opening the small window

This leads to the next step in preparation, which is the exact execution trigger I had to decide on. This can be a little tricky to figure out in general, but even more so in this case because of the challenge period. If we're using a chain with synthetic UNI, the jar value should always exceed the current UNI market value since we'd be able to easily bridge the UNI back and sell on mainnet. It's not as simple in this case. Moreover, if the price of UNI is dumping, you'd also rationally want to claim a jar that at least covers the cost of all the UNI you bought. In this case, since I was ~delta neutral and hoping the UNI price would dump, I needed to ensure that claiming the jar and closing the short would leave me better off than waiting seven days to bridge back and unwind the short. To add to the complexity, I am also perma-long ETH and had no interest in taking an ETH loss if I made a USD profit, so my trigger was as follows:

```java
pool coverage is complete to ensure an accurate jar valuation
AND
at least break even on an ETH basis if the jar+hedged position is liquidated to ETH
AND
at least break even on a USD basis if the jar+hedged position is liquidated to USD
AND
wait for up to $420.69 profit unless a credible competitor shows up
```

There was also an emergency policy where I was willing to sacrifice some money just in case a competitor appeared, up to the funding fees of my short:

```java
credible competitor detected
AND
original ETH principal remains protected up to a small economic sacrifice
AND
the economic sacrifice is no greater than accrued hedge funding (~$50)
```

This meant that in the worst-case claim scenario, I'd just lose up to about $50 from the ETH cost basis, a price to pay to bridge back and free the rest of my capital. In the best case, I wanted up to $420.69, which was still typically smaller than a lot of jar claims. If I missed the claim altogether I'd just bridge back and take the loss of time on the chin. The final end-to-end pipeline looked something like the following:

![pipeline](/assets/img/sample/tokenjar/pipeline.png)

### Executing in prod

The UNI market started to dump, and it started to validate that the short was a successful thesis. Since I was ~delta neutral, this didn't generate any profit for me directly. However, being ~delta neutral means I could evaluate the jar's profitability against the combined spot and short position. This meant that as soon as the jar value exceeded the *current* cost of the required UNI plus the short proceeds, I was in profit (bar infra costs and misc on-chain costs). During severe market swings, trading volume picks up as well, so even more trading fees flow into the jar, as can be seen in the chart below:

![window](/assets/img/sample/tokenjar/window.png)

As the price was dipping, I started falling into the execution window. At this point, I was ready to manually intervene since there was some non-zero profit and I really had no interest in waiting several more days for the next jar or needing to wait out the challenge period. However, since there were no competitors coming in, I let it run. After some more market turbulence, I saw we'd hit our threshold and I expected the trigger to hit:

![above](/assets/img/sample/tokenjar/above.png)

Except...it didn't work. We hit every gate in our normal trigger but things were failing. The repeated pool scans took minutes and the local vs remote pricing drifted apart so it became unclear if the trigger was just a false positive. Moreover, the tx required >19 million gas due to the additional leg of liquidating claimed assets, which was more than the 16.7 million per-transaction gas limit on Optimism. I was pretty heated since I assumed things would be smooth sailing after learning from several failed attempts, and I felt like I had no time to debug it since the market reversed fast and the opportunity could disappear. To make matters worse, my Telegram bot pinged me to tell me there was a competitor on-chain too, and it meant my opportunity was about to slip if I didn't act **fast**.

![tg](/assets/img/sample/tokenjar/false-pos.png){: width="400" }

At this point, I fully switched into emergency mode. I asked Codex to run the backup script to call the `UniFeeClaimRouterOptimism` contract and I began running my local emergency scripts just to have either one of them land. Just to make things even more dramatic, **both** of these scripts failed too, and for different reasons! The local emergency script used the stale pricing and did not pass every trigger gate, so no tx was signed and broadcast. The remote emergency claim passed every trigger gate, but like before, the tx exceeded the gas limit despite the liquidation leg being excluded.

![stopped](/assets/img/sample/tokenjar/stop.png){: width="400" }

Eventually, during a heightened sense of urgency, Codex repackaged the tx without 25 of the lowest-value pools, which was sufficient to reduce the gas usage down to 14.46 million. With great relief, the repackaged transaction that flushed 275 V3 pools, 18 V4 currencies, burnt 2000 Optimism-bridged UNI and claimed 20 token balances landed on-chain:

![claimed](/assets/img/sample/tokenjar/claimed.png)

A simple Codex prompt to liquidate the claimed assets was trivial and I market closed my UNI short on Hyperliquid (for a cool 17.5%!) to avoid any further one-sided risk.

![jar mech](/assets/img/sample/tokenjar/closed-short.png)

Testing in prod is stressful, fun and maybe even profitable if you're persistent. But I do *not* recommend it. There are many more ways to make money on-chain, and you might even find some ways to do it without losing your hair.

## Dual-basis PNL

I think it's worth briefly talking about the PNL against both ETH and USD. Of course, having just one target PNL (the smaller of the two) would increase the likelihood of claiming the jar. In hindsight, the short helped reach the goal of hitting my cost basis in both ETH and USD because of the sharp downtrend. I vibe-charted what an unhedged position's growth looked like, and it's clear that the short allowed me to claim the jar earlier xor at a more profitable level:

![jar growth](/assets/img/sample/tokenjar/jar-growth.png)

Of course, it's not clear if your competitor is ever hedged since they may be participating on any perp DEX at some other address, or even a CEX. In any case, I am lucky that there was a downtrend instead of an uptrend. In the case of an uptrend, my short would lose money, and it would have been more likely that an unhedged competitor would come and claim the jar. It would be more profitable if they had bought before the rise and were measuring profit against their entry cost.

While I don't care to talk about PNL for the sake of it, I think it's interesting to talk about some numbers just because of all the dynamics involved. I targeted `+$420.69` in profit while at least covering my ETH basis, but since the claim was delayed due to the failing initial execution attempts, the profits ended up being higher than the target. The gross returns and costs are as follows:

- USD return: `+$1,617.79`, or `4.61%` profit on the combined capital committed to the UNI spot position and the short, denominated in USD.

- ETH return: `+1.3194` ETH, or `10.14%` profit over combined UNI spot denominated in ETH + short USD capital denominated in ETH at the time of each transaction. The `%` return is higher than the USD denom profit because it accounts for the ETH downtrend during the project. 

- UNI short return: `+17.5%` from entry (`$8.822893`) to close (`$7.2753`), so approx `$3,064.70` gross price P&L, or `$3,110.76` including funding. Of course, this is offset by a similar-sized loss due to the decline in the spot position's value.

- Transaction and bridge costs: `~$23.20`

- Hyperliquid trading fees: `~$9.12`

- Infrastructure: Codex `$100` + Chainstack `$25` + DigitalOcean `~$14` = `$140` total

Overall, while it was profitable, I think the actual return was not worth it given the risk. There were so many vectors that could have gone wrong, such as bridging risk, an incorrect nonce used for the claim, incorrect assets claimed through bad coverage, etc. Compared to all other long-tail MEV strategies I've run, this may have been profitable in absolute terms, but in relative terms it's the worst performance I've had just due to the type of opportunities I look for. Typically I look for strategies that allow me to profit with no/low capital involvement, low competition and high % returns. This entire strategy turns my preferences on their head with high upfront capital required, high competition and much lower returns.

## Capture postmortem

There's a lot to learn from this entire process. I'll highlight some of the noteworthy things that went wrong during the actual production execution, since that's what is the most relevant and something you could learn from for your own searching. Of course, it wasn't all bad! We got a valid claim with some money for lunch. Here are some things that went well:

- **The UNI short worked well:**

    Having a ~delta neutral UNI position was a good idea to keep us ahead of unhedged potential competitors who might have bought UNI before the downtrend, and also protected me from real losses had I just kept the spot UNI as is. I believe this is the main reason I was able to profit over both USD and ETH cost bases, which I'm satisfied with given the circumstances.

- **Accurate jar valuation led to a profitable tx:**

    After several rounds of bad coverage, I eventually converged on good coverage and good jar valuation, which means by the time the tx landed I was quite sure we were in profit despite not having the atomic liquidation leg.

- **The emergency backup (eventually) worked:**

    Part of the beauty of defi is the composability aspect, and being able to use someone else's permissionless contract was a cool way to get my claim through, even if it was quite annoying/stressful that my deployed contract didn't work.

- **Enumerating potential competitors was a great idea:**

    Going through all the UNI holders on the chain and trying to discern who might be interested in the jar, and finding none, really helped alleviate my stress and gave me the last push to see this project through. Of course, the system was not perfect and in hindsight there may have been other ways a competitor could have cropped up, but it was a good implementation regardless.

Now for some of the things that went wrong:

- **Stale pricing and slow scanning almost cost me the jar:**

    - Something I didn't have during the entire process was a paid-for API/price oracle of some sort. The system was just collecting data from free-tier APIs because, again, I underestimated the size of this undertaking. Moreover, price feeds tend to get stretched during market turbulence which is part of the reason we landed up with stale pricing when we wanted to execute.

-  **Emergency backups failing during an emergency:**

    - This is a tough point, and it's clear that asking Codex to "test the scripts for robustness" and asking for its "confidence level" on execution probability is just completely insufficient. (I'll speak about this more in the next subsection.) Nevertheless, these failures can now be iterated on for further claims.

- **False competitor alert caused panic:**

    - While monitoring was definitely a good idea, a bug in the script that caused it to flag a competitor when there wasn't any was extremely unfortunate. This caused me to hit my emergency scripts immediately, and while I could make out that this was likely just a mistake during the chaos of it all, I still chose to believe it might be true. I could have chosen to let things ride even more and let the jar collect even more profit, but this wasn't my priority at the time.

- **Double the capital needed for the hedge:**

    - I can immediately acknowledge that being able to go delta neutral while stuck behind a potential 7-day window is a very lucky position to be in. Beyond that, having capital locked up that isn't producing money beyond funding fees is less than ideal.

There are several more points for either side of the coin, but this is the gist of the postmortem.

### Thoughts on blind AI steering

I used Codex with daybreak/sol models for the project, and otherwise manually edited small scripts or minor configs here and there. On one hand, it's quite cool that with the correct series of prompts I can essentially print money out of the computer and cover the cost of the infrastructure used and my Codex subscription. On the other hand, things went wrong often, and without my oversight and experience in defi so far, I could have very easily burnt my UNI along the way or encountered a similarly devastating consequence. I have some takeaways from using AI for this entire project:

- Don't flip flop your real money in between chains with out a small test! This should be obvious, but if you're trying to be competitive it can be tempting to just yeet your money and hope that your agent turns a profit for you. This is especially true for more specific things like minting synthetic UNI, for example.

- Getting complacent shows quickly. The entire delta-neutral hedge portion of this blog post could have been avoided had I actually gone and read up about the UNI required on Optimism. Instead, I assumed it was a similar situation to the Wormhole bridge that can teleport the UNI back immediately, which is what I can only call a moment of weakness/complacency in my urgency.

- Be explicit for each prompt. Prompts like "think about what could increase our pool coverage" are vague and not helpful, even if the goal is actually to increase the pool coverage in the scans. I may have prompted something like this 3 times over the course of the project but it wasn't until the failure in production that we discovered >100 pools were missing from our coverage because they were included in the genesis state. Prompting for evidence, reviewing what was tested and checking whether the backup *actually works* are simple points to start from.

I feel quite fortunate to have been using/hacking/testing defi protocols long before AI became so widespread. Across the long-tail searching I've done, I've written scripts entirely by hand and managed the entire infra stack by myself, and I've also [written strategies with AI assisting me](https://ashiq.co.za/posts/Arbitrum-latency-game/) along the way. And now, I've been able to have one instance of a profitable strat (nearly) fully managed by AI. Personally I think writing everything by hand is a bit tedious, but having AI manage everything turned out to be much more stressful because things were just unreliable at the time of execution. Of course you could go ahead and read all the code that your agent is writing for you, but it's not something I did given that I thought this was a small project to piece together and wrap up. I think the middle ground is the best option: piece together your scripts, contracts, etc., but have AI assist *you* doing it, so you have good visibility over your pipeline, while avoiding the unnecessary overhead of threading together lots of syntax. It would be easier to port this project to the middle ground, and I can be a lot more certain on execution should I want to capture more jars in the future.

## Conclusion

Overall it was a fun project, and one that I severely underestimated the size of compared to when I first started. Hopefully you've learnt something from this for your own searching.

Also, I've been a solo searcher/researcher for some time now and I feel ready to join a team in a more formal capacity. Ideally I'd join a team doing similar research related to long-tail MEV or anything adjacent to this blog post. My twitter/Telegram (@AshiqAmien) dms are open, feel free to reach out! :)

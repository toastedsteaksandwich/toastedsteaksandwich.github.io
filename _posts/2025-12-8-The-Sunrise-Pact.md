---
title: 🌞🌱 Aligning incentives with the Sunrise Pact
author: Ashiq Amien
date: 2025-12-11 13:25:00 +0200
categories: [Research, MEV]
tags: research
math: true
---

## Intro

Around a year ago I had an idea for a long-tail MEV strategy built around hourly Bean emissions from the [Beanstalk protocol](https://bean.money/). What started as an experiment turned into a year-long attempt at capturing MEV by coordinating with pseudonymous on-chain actors. This blog post walks through the timeline of the strategy with some of the insights I picked up along the way. 

**TL;DR**

- Beanstalk emits hourly Bean rewards, first caller to claim takes all.
- Two or more actors can mutually benefit by coordinating on-chain instead of competing.
- The coordination game looks very similar to a repeated Prisoner’s Dilemma.
- The game is vulnerable to players introducing additional Sybil identities.
- The strategy worked until dilution and incentive misalignment broke it.

### **Motivation**

I was trying to learn about MEV searching when I came across [this video](https://www.youtube.com/watch?v=NqM9PNPQFMs) where [Anish Agnihotri](https://x.com/_anishagnihotri) mentions early searchers were able to earn [~$100 per hour by calling a simple function](https://youtu.be/NqM9PNPQFMs?si=FEwXVAClzl-B8bzP&t=1269) on the [Beanstalk protocol](https://bean.money/). This meant that, in theory, a searcher could print nearly $850k per year for doing... close to nothing? Of course, it's not 'nothing' since you'd need to ensure uptime of your box, always have enough for gas, not get any competition for the entire year, etc. But considering the R/R of nearly $1M for a simple task, I decided to investigate things for myself. Unfortunately, the Beanstalk protocol was much older than the time Anish was talking about and had already been [superseded](https://x.com/bwein_/status/1953103625779032089) by a new stablecoin protocol [Pinto](https://pinto.money/) deployed on Base. Further, Beanstalk's hourly rewards were [reduced considerably](https://github.com/BeanstalkFarms/Beanstalk-Governance-Proposals/blob/master/bip/bip-30-generalized-pipeline-permit.md), and the actual Bean price depegged far from the [dollar](https://www.coingecko.com/en/coins/bean). Nevertheless, I spotted that there was just one actor, 0xc1c, silently crunching away at the same task on Beanstalk Arbitrum receiving rewards around ~$5/hour. Despite the value being 95% less than the original amount mentioned in the video, there were 3 things about the situation that got me even more interested:

- There was just one actor, meaning that there was no competition
- Beanstalk migrated to Arbitrum, so the gas was cheap compared to Ethereum mainnet
- Arbitrum has [no frontrunning](https://docs.bean.money/developers/misc/faq#who-calls-sunrise), so generalized frontrunner MEV bots are less of a threat for this scenario

I had an idea to capture some of those rewards for myself without destroying the competition. Before explaining the idea, I'll flesh out a little about how Beanstalk works a bit more and what the task is.

### **Bean infrastructure**

Beanstalk is a decentralized stablecoin protocol on Arbitrum (previously on Ethereum mainnet) with Bean as its USD-denominated stablecoin. The full mechanics of Beanstalk can be found [here](https://docs.bean.money/developers/overview/introduction). Our main interest is the [Sun](https://docs.bean.money/developers/protocol/sun), where the docs describe the Sun as its timekeeper:

![Desktop View](/assets/img/sample/sunrise-pact/the-sun.png)

The part we're interested in is the hourly `gm()`/`sunrise()` functions, i.e. the simple task mentioned in the motivation. These functions have the following properties:

- `sunrise()` and `gm()` are timekeeping functions and either one needs to be called every hour, on the hour with up to 5 min delay. `sunrise()` needs no parameters and `gm()` needs params to specify the reward destination and transfer type, but they're effectively the same beyond that.
- The rewards from the call goes to the first caller only and no further rewards are emitted from the Sun until the next hour.
- The gas fees of the call are low on Arbitrum during normal traffic (~$0.01-$0.15), but could increase to several dollars during times of congestion.
- The rewards from the call starts at 1 Bean and compounds 1.0201% every 2 seconds for 5 minutes for a max reward of ~19.79 Beans, as follows:

$$ y = (1.0201)^{x/2} $$

![Desktop View](/assets/img/sample/sunrise-pact/graph2.png)


I'm going to use `gm()` going forward in the blog, but this is interchangeable with `sunrise()`. Given the above, the intuitive play to start farming rewards seems obvious:

1. Look at who's calling `gm()` and at what time
2. Prepare a call to `gm()` at slightly less seconds than the competitor 
3. Wait until the next hour and repeat (1) if outpaced by another competitor  

While this will earn you _some_ rewards, it should be clear that with an active competitor, you would drive quickly your hourly rewards down to the minimum reward as they would likely mirror the same steps as above. The important part here is that by undercutting the competitor, you _both_ eventually end up getting lowered rewards after several rounds of competing. Keep in mind that the rewards compound in the 5 minute window, so even just slight competition reduces your rewards considerably.

### **The first pact**

There was just one active actor, 0xc1c, calling `gm()` so I came up with an idea for us to come to some kind of agreement to split rewards so we don't have to compete. By making a smart contract that splits the reward for both of us, I'd be able to hold back from undercutting the rewards and preventing the downward race to 0 seconds (i.e. driving the sunrise time down to 0 seconds, meaning only 1 Bean is emitted). This would allow both of us to farm Beans at the max rewards making it worthwhile for both of us to cooperate long-term. Moreover, since we live in *permissionless-free4all-blockchain* land, I don't need to ask anyone if I can write the code and plug into the protocol. I can just deploy a smart contract and try to convince 0xc1c that my proposal is worth agreeing to. I wrote a basic smart contract, called the _Pact_, and an excerpt is shown below:

```javascript
modifier onlyParties() {
    require(
        msg.sender == you || msg.sender == me,
        "Only authorized parties can call gm"
    );
    _;
}

//just trying to get your on-chain attention!
function attention() external {
    require (msg.sender == me);
    uint256 sunriseBeans = beanstalk.gm(me, 1);
}

function gm() external onlyParties {
    // Call Beanstalk's sunrise function and get beans earned
    uint256 sunriseBeans = beanstalk.sunrise();
    //just in case
    uint beansReceived = bean.balanceOf(address(this));
    // Calculate shares        
    uint256 callerShare = (beansReceived / 2) + TX_FEE_REWARD;
    // Transfer beans + fee to the caller        
    bean.transfer(msg.sender, callerShare);
    // Transfer remaining beans
    bean.transfer(msg.sender == me ? you : me, beansReceived - callerShare);
    emit thx(msg.sender, beansReceived);
}
```

The code is straightforward: the new `gm()` function is a wrapper that calls `sunrise()`, then receives the minted Beans from the [Beanstalk diamond](https://arbiscan.io/address/0xd1a0060ba708bc4bcd3da6c37efa8dedf015fb70) (i.e. the protocol entry point), passes on a small tx reward to the caller and lastly splits the remaining Beans. The `TX_FEE_REWARD` is sent to the caller to compensate for gas. I thought this was necessary since Beans were already depegged, so some compensation was needed to cover gas costs to be fair. 

The last piece of the puzzle is to signal to 0xc1c that I'm not looking to compete and that they have the option to switch their bot to point the address of the pact contract. Before talking about signalling, it's worth asking why a competitor would want to cooperate at all, given that I'm proposing they give up _half_ their rewards indefinitely.


### **On-chain Prisoner's dilemma**

After doing some vibe-researching, it turns out that this coordination problem is a variant of the [Prisoner's dilemma](https://en.wikipedia.org/wiki/Prisoner%27s_dilemma) game theory experiment. With our pact, we have the following truths:

- Both actors benefit if we cooperate (~19.8/2 Beans each)
- An actor individually gains more Beans by undercutting the other (~19.7 Beans for the undercutter, 0 for the other actor)
- A race to call `gm()` at 0 seconds is worse for everyone (1 Bean for one actor, 0 for the other)

We can tabulate this as follows:

![Desktop View](/assets/img/sample/sunrise-pact/game-theory.png)

Given the above, we see that both players have power over each other since their actions directly affect one another. For both, it's in their best interest to agree to cooperate to farm max rewards (at 300 seconds) and not undercut each other to avoid a race to 0. Of course, it would be better for the first player if there was no competition at all, but this is a permissionless winner-takes-all game that happens to be available to anyone. As soon as *any* party knows about the game, they inherit the same power over all players. 

There are some other angles that make this game more interesting than vanilla Prisoner's dilemma:

- Players are pseudonymous users on a blockchain, making the game vulnerable to Sybil attacks. There's no way to prove a 3rd person is not already a participant in the pact.
- The blockchain is transparent, and the pact code is public. If it wasn't public, it would make it harder to convince other players to cooperate. 
- The pact might be permissioned, but the Beanstalk protocol is not, allowing anyone to undercut at any time. 
- The game is repeated hourly, and it's not just a once-off reward or punishment, skewing the long term incentives.

### **Signalling**

Let's loop back to signalling. Once the pact contract was deployed, I was wondering how to communicate my proposal. Arbitrum is a busy chain and 0xc1c probably won't see the deployment unless they have monitoring tools or they're glued to the explorer. I figured the closest way to send a message would be with a [UTF8 encoded message](https://medium.com/@Carsonated/how-to-message-via-ethereum-tx-14ae54487eee) and just hope it lands:


![Desktop View](/assets/img/sample/sunrise-pact/initial-signal.png)


I sent a few of these messages and alongside 2 other signals. The first signal being calls to the `attention()` function described in the code snippet earlier in this post. The idea here is to call this function at 0 seconds to completely undercut the competition, and doing so several times with the hopes that the competitor will look at the explorer and try to see what the latest sunrise time was. The other signal I sent was calling the deployed pact and splitting the reward optimistically, hoping the competitor would pick up that I'm trying to work together.

I was hopeful that a combination of these signals would eventually reach. Calling `gm()` through the deployed pact means the tx doesn't show up on the actual [Beanstalk diamond](https://arbiscan.io/address/0xd1a0060ba708bc4bcd3da6c37efa8dedf015fb70) directly, needing the competition to dig through the explorer and eventually find my contract source with the proposal to coordinate (at least, that's the way I usually dig on explorers).

It became clear that the competition had written a bot that reacted to the latest `gm()` calls without manually viewing the explorer. After consistent `gm()` calls at 5 min from 0xc1c, any undercuts I performed was met with further undercuts until we raced to 0. Nevertheless, after about 3 weeks of signalling, there was a breakthrough:

![Desktop View](/assets/img/sample/sunrise-pact/signal-accept.png)

Sorted! I was elated to find that the blind signalling actually worked. Hourly incentives were coming through thanks to on-chain coordination. The story does not end here though!

### **Maintenance, iteration and progression**

As far as I remember, the next few months were relatively smooth. Some notable events were:

- `TX_FEE_REWARD` updates: 

    The first iteration of the pact set the `TX_FEE_REWARD` to 1. This was a mistake since it left the caller with 2 extra Beans to compensate instead of just one. Changing the reward down to 0.5 was trivial, and 0xc1c switched to the contract with the correct parameter.

- Changes in my tx execution:

    During the signalling stage I used a [rust program](https://gist.github.com/AshiqAmien/c0d0d7b584a2a2d04796686df56281df) to do my `gm()` calls. It was overkill but it was just an adapted program from my [backrunner bot](https://ashiq.co.za/posts/Arbitrum-latency-game/). This was sitting on a Hetzner box, but since I opted to not be the caller, I eventually switched to a one line cron (i.e. `cast send "gm()" <address> <key> <rpc>`) on my local machine. This was run loosely since 0xc1c's bot seemed fairly reliable, and my cron was configured to call `gm()` _after_ 5 minutes in case their bot failed for some reason.

- Occasional undercuts due to congestion and strays

    I was surprised to learn that Arbitrum gets congested often. During times of high congestion and subsequent high gas prices, 0xc1c opted to call the Beanstalk diamond directly and take on the full reward to compensate for the high gas fee. Usually the reward would outweigh the fee, but I wasn't motivated enough to write + host a bot just for this given that I already took down my Hetzner box. There were also occasional stray actors that came in and undercut both of us, but none of them seemed to stick around for too long. 

It was only further down the line, around 7/8 months in where a third competitor came in, and stuck around. 

### **Beans, 3 ways**

The 3rd actor, 0x37b, came in with an interesting twist: they sent over a UTF8 encoded message to 0xc1c with their tg handle. I monitor the explorer often so I picked up on this and sent a dm:

![Desktop View](/assets/img/sample/sunrise-pact/hello.png)

I'm not going to leak any of the conversation since I don't believe leaking dm's is professional, but a gist of what we discussed could be summarized as a negotiation of a new pact. From a game theory angle, there's a third person in our prisoner's dilemma, which means our cooperation dynamics looks a bit different now. Here's what the new coordination outcomes table looks like for 3 farmers:

![Desktop View](/assets/img/sample/sunrise-pact/3-player-table.png)

An interesting dynamic gets replayed here: you don't need a lot of capital or need to know any insiders to get your share of the pact. It's even simpler: all you need is to know is that the game **exists**, and that your power in negotiation is equal to all other participants in the game. That is, you're at mercy of the other 2 players to include you, while the other 2 players are at mercy of you to not undercut or race to 0. Unfortunately, as long as the new player is persistent, your only two real moves are:

- Burn out the competition by racing to 0 and staying there until someone gives up.
- Include the new person and take the dilution to the chin.

Given that things were already relatively smooth, I was ready to accept the dilution. I didn't think burning out the competition was a smart move given that this is already a vulnerable strat and I guessed it probably wouldn't be long before even more players want to join.

### **We don’t allow Sybils in this house!**

After some negotiations, 0x37b created their own pact that split the rewards 3-ways equally. Given that I was ready to accept the dilution, I sent a UTF8 encoded message to 0xc1c to signal the new pact, and a few days later, they started to undercut and race to 0 seconds. This lasted nearly a month before 0xc1c deployed a new, revised pact:

![Desktop View](/assets/img/sample/sunrise-pact/new-3-pact.png)

I started growing suspicious that the original pact was being [Sybil attacked](https://en.wikipedia.org/wiki/Sybil_attack). For one, 0xc1c was not cooperating for nearly a month, and then releases a new pact that includes a small, but unequal, portion for 0x37b. Moreover, the updated pact used the same template as the one made by 0x37b despite had never been used for coordination by that stage. Lastly, I noticed that both 0xc1c and 0x37b called the beanstalk diamond directly when Arbitrum was congested, meaning they both had bots that consider the gas price before the `gm()` call:

![Desktop View](/assets/img/sample/sunrise-pact/double-gas.png)


Maybe I was just lazy and other searchers have template bots that already consider the gas price on Arbitrum tx execution, while I didn't. Either way, not long after I noticed the above, 0x37b deployed an updated pact with new proportions:

![Desktop View](/assets/img/sample/sunrise-pact/final-3-pact.png)


I spent time thinking about a Sybil attacker's perspective and realized that trying to burn out the competition might have been a smarter move than accepting dilution like I did. Trying to include a Sybil account into the pact might look something like the following:

1. Undercut the competition for a long time from the to-be-included Sybil account and hope that the other party (me) gets uninterested and forgets about the pact. If this works, get the full ~19.8 Bean reward going forward. 
2. If (1) doesn't work, begin negotiations and attempt to include the Sybil account, while communicating as the Sybil account. 
3. If (2) doesn't work, switch to the original account and add the Sybil account to the pact but only with a small portion of rewards. Cooperate for some time and then increase the portion until the Sybil has an equal portion. 

The outcomes from the above steps incentivize attempts to burn out the competition. The risk is guaranteed short-term loss for potential long term increased rewards (50% -> 66.6% is a 33.3% proportional increase in rewards). Based on the thinking outlined above, I came to the conclusion that 0x37b was a Sybil account of 0xc1c and decided I needed to push back. In hindsight, my conclusion was likely incorrect, although there's no definitive evidence either way.

### **The timelocked escape hatch**

After being diluted I tried to think about some ways I could push back. I drew out 5 options:

1. Undercut at 4 min 59 sec to claim most of the reward myself, but only do this periodically
2. Call the original pact between myself and 0xc1c to signal I'm not interested to share rewards at the current proportions
3. Make a new contract with my preferred pact proportions and call that
4. Burn out the competition until someone becomes uninterested
5. Do nothing

I decided on option (1) with (2) as my second option. Before being diluted I was getting ~200 Beans/day, and after being fully diluted I was at ~130 Beans/day. If I wanted to get back to the same Bean rate with (1), I would need to undercut 5 times per day: 

```
(5.7 * 24) // diluted Beans/day
+ (19.7×5) // undercut Bean rewards * 5 
− (5.7×5)  // less 5 rounds of diluted rewards
= 206.8    // final Beans/day
```
 
5 rounds of undercutting felt too frequent given that the competitor bots react immediately to any undercuts, so I just started with 1-2 per day. 

Over time, this method of undercutting didn't seem to work well since there were several stray-looking bots coming in and undercutting and racing to 0. Eventually once the strays disappeared and things started to look stable, I decided to take on action option (3) and update the pact to cut out 0x37b entirely, along with other additions to further incentivize cooperation:

```javascript
function gm() external onlyUs {        
    uint beansReceived = beanstalk.sunrise();
    BEAN.transfer(msg.sender, TX_FEE_REWARD);           
    if (flag){escapeTime = block.timestamp + 3 days; flag = false;}    
}

//if there is at least 100 beans in this contract, split everything evenly 
function withdraw(BERC20 token, uint256 amount) external onlyUs {
    uint beansReceived = BEAN.balanceOf(address(this)); 
    require(beansReceived > 100e6); 
    BEAN.transfer(me, beansReceived / 2);
    BEAN.transfer(you, beansReceived / 2);          
    flag = true;    
}

//if you don't agree, sweep everything after 3 days
function escape() external onlyOwner {
    require(block.timestamp > escapeTime);
    uint beansReceived = BEAN.balanceOf(address(this)); 
    BEAN.transfer(me, beansReceived);
    flag = true;    
}
```

The main feature of the new pact was an _escape hatch_ used in the case of the other party not cooperating. If the Bean threshold was not reached 3 days after the last sunrise, I'd be able to withdraw the full balance. The idea here is that I'd be able to sunrise 2-3 times and let the balance sit on the contract until the other person cooperates. This allows the reward to be split evenly under ideal conditions or otherwise leaving me with the escape hatch in suboptimal cases. This allows me to get my reward proportion I want (and undercut while waiting), except in races to 0.

It wasn't long before I spoke with 0x37b on telegram again who claimed it was not a Sybil attack, and that there was no fake dilution. It also wasn't long before 0xc1c sent a final UTF8 encoded message claiming similarly:
 
![Desktop View](/assets/img/sample/sunrise-pact/sunrise-dead.png)

While I still have no 100% proof it wasn't a Sybil attack, I have a feeling there was no fake dilution. My main blind spot was that I was the first person to propose the dilution altogether, making the entire scenario a lot less likely. Further, once I started growing suspicious, I started checking the explorer more often for subtle cues on collusion. It's easy to spot these if you're looking out for them, but ultimately there's no real visibility on what's going on. 

### **Garnish**

Since the final pact, there's been a race to 0 and no sign of recovery since then. Did I kill the strat? Possibly. Would I have done it differently? No, since I felt like it made sense to fight for my original proportion. I have high doubts that my proportion will come back, especially since this entire post opens the pact for complete dilution and no recovery (of course, I considered this aspect for some time before writing+posting). There's some final extra things I wanted to mention that didn't belong under any main heading:

**Costs**: I typically avoid talking about PNL in MEV/bounties where I can since I feel it distracts from the content, but I thought it was worth mentioning here. Overall, I really appreciated the low barrier to entry for this long-tail strat. Hetzner nodes cost ~5 EUR a month and gas on Arbitrum is very cheap most of the time. I estimate I spent a total of ~50 USD total over the year, but even if you were completely strapped for cash, you could probably have been as successful with less, maybe even enough for just gas. Having money for gas means you could undercut once and sell your 19 Beans, leaving you enough money to deploy your pact and undercut a few more times. With pact cooperation you could cover your Hetzner node monthly cost 2 hours. Or if you're very cash strapped- you didn't need a node at all! A single line cron on your local machine while you're awake would have sufficed when everyone was cooperating. This is in contrast to other long-tail strats I ran this year where I had to pay for expensive infra, only for the strat to die within a week or two.  

**Bean price/Bean rev**: The [Bean price](https://www.coingecko.com/en/coins/bean) was very unstable throughout the journey. I don't know the full mechanics of how the Bean price is meant to stay pegged, but I noticed that the Bean price typically floats with the ETH price. To measure the success of the strat I thought the best way is to measure the Beans accumulated over time and not the ETH value. To do this I used the awesome [Envio's HyperIndex](https://envio.dev/) tool to pick up on the relevant transfer events and vibe-charted the cumulative Bean income over time, overlaid with the ~daily Bean income:

![Desktop View](/assets/img/sample/sunrise-pact/bean-chart.png)

## **Conclusion**

Overall this was a very fun long-tail strat, it's definitely the longest running one I've had. As far as I can tell, this hasn't been done before so I've been excited to write this post and publish a bit of what I've been up to. I hope this inspires other on-the-fence searchers to just get started with checking the chain and realizing you don't need crazy assembly skills or big $$$ to start!

**If there's any other solo searchers looking to collab or any MEV shops looking for an intern, please reach out! My twitter DM's are open** :) 

![Desktop View](/assets/img/sample/sunrise-pact/closing.png)
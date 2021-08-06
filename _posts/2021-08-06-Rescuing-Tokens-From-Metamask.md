---
title: 🧑‍🔧 Rescuing tokens from a compromised Metamask with Flashbots
author: Ashiq Amien
date: 2021-08-06 10:00:00 +0200
categories: [Research, Flashbots]
tags: research
---

## Background

I recently received a message from a high school friend asking for help regarding his compromised Metamask:


![Unfortunately I'm not a night owl anymore](/assets/img/sample/flashbots-rescue/help.jpg){: width="350" height="300" }

The wallet was compromised because he was phished into sending his Metamask seed somewhere. The phisherman drained [163 xSUSHI](https://etherscan.io/tx/0xd3f95d70470689d60ffe643207fdfe5b98a4e4a45228a1e11bb37aa908336859), worth ~$1.5k at the time, as well as the [remaining ether](https://etherscan.io/tx/0xc9ff6939eb8981008120e0a1bad175ef6435f1b9562e18b0c3926df7b5b9b1c5), worth a few dollars. There was still ~$8k worth of Sushiswap WETH/WBTC LP tokens in the wallet that the attacker had not stolen. In an attempt to retrieve the LP tokens, my friend sent some extra ETH to the wallet, which was immediately swooped up the attacker after a few blocks:


![Swooped ETH](/assets/img/sample/flashbots-rescue/eth.png)

Unfortunately, I did not get around to seeing any of this as I only read the messages several hours later when I woke up. Surprisingly, the attacker had not done any further damage, leaving behind the LP tokens, 14 CryptoKitties and 1 Polkamon. By this stage, it seemed like the attacker was not interested in the LP tokens for some reason - I imagine that either a bot was being used to sweep the wallet for Ether and whitelisted tokens, or the denomination of the token seemed so low that it was unlikely to be valuable. Since it wasn't gone in 7+ hours, we figured it would be fine to wait until I could go around to his place to try and recover the remaining tokens. 

Over the last few months I read several stories about using [Flashbots](https://docs.flashbots.net/) auction to rescue tokens from compromised wallets, and it semeed to fit into this scenario perfectly. Flashbots auction is a way to communicate to miners directly to ask for your transaction (or transactions) to be included into a block, bypassing [the need to go through the mempool](https://samczsun.com/escaping-the-dark-forest/) as usual. The incentive for a miner to include your transactions in a block is paid directly to the miner in the form a bribe. Unlike normal accounts, this tip can be paid for by any other account. This is exactly what we need - we can siphon out the Sushiswap LP tokens and pay the bribe from a separate account, because assume any ETH sent to the compromised wallet will get swept.

### Using Flashbots to rescue ERC20 tokens and CryptoKitties


To get started, I tried building a local version of [flashbots.tools](https://flashbots.tools/), which was aGUI to interact with flashbots, as well as [@smpalladino's tools](https://twitter.com/smpalladino/status/1373312670812807174) used for the same thing. Unfortunately, neither worked at the time since the tools did not work with the new `jsonrpc` 2.0, and fixing the issue seemed like it would take a while since I was unfamiliar with the setup. Luckily after looking around on discord, I found [searcher-sponsored-tx](https://github.com/flashbots/searcher-sponsored-tx), which seemed to do the same thing.

We set up a fresh metamask, funded a wallet with some ETH for the bribe and populated our payload with everything as per the instructions. This included the compromised private key and the key of the donor wallet. I used Alchemy API to interact with the Ethereum mainnet, and `relay.flashbots.net` to interact with the relay's RPC endpoint. After some last tweaking, we ran the tool and the tokens were [sent to our own wallet](https://etherscan.io/tx/0xa694201e011663de0dfc88577e480906210de9806487a53a409ab6b1bffbb070) 🥳:

![Notice the 0 gas price](/assets/img/sample/flashbots-rescue/done.png)

There were also some CryptoKitties that needed to be rescued, which I got round to a few days later. By the time I got round to this, EIP1559 was live and `searcher-sponsored-tx` was updated to work with the new gas pricing mechanism. Setting things up was equally straightforward, with the only tweak needed to be made on the `index.ts` file so that the CryptoKitties engine is used:

```
const KITTYIDS = [31337,42069];
const engine: Base = new CryptoKitties(provider, walletExecutor.address, RECIPIENT, KITTYIDS);
```

This worked smoothly, with 7 CryptoKitties now rescued 🐈: 

```
[2021-08-06T09:28:24.550Z] Sending these kitties 741607,768114,433725,769361,769468,623040,769422 from 0xB2e701bC259bBe5Fdb76123ba1c0ceC78C46A103 to 0xC44F1B936F0C0bcf691D6d05057218425241Bb1d
[2021-08-06T09:28:24.551Z] Executor Account: 0xB2e701bC259bBe5Fdb76123ba1c0ceC78C46A103
[2021-08-06T09:28:24.551Z] Sponsor Account: 0x63dCEE589e58EcA4E1FDD04C58e5827040CC3609
[2021-08-06T09:28:24.552Z] Simulated Gas Price: 35.07 gwei
[2021-08-06T09:28:24.553Z] Gas Price: 64.88 gwei
[2021-08-06T09:28:24.553Z] Gas Used: 486801
[2021-08-06T09:28:27.620Z] Current Block Number: 12970606,   Target Block Number:12970608,   gasPrice: 35.07 gwei
[2021-08-06T09:28:50.485Z] Congrats, included in 12970608
```
![Rescued CryptoKitties](/assets/img/sample/flashbots-rescue/kitties.png)

### Conclusion

Before this incident, I was meaning to try out flashbots auctions on a test wallet, but I just never got round to it. I think this was a cool way to get introduced to it, but there's a lot more to learn from here. If there's anything we can learn from the above, it's to make sure you *never* send your wallet seed, anywhere, ever.


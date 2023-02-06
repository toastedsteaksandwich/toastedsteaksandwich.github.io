---
title: Research and Training

# The Research page
# v2.0
# https://github.com/cotes2020/jekyll-theme-chirpy
# © 2017-2019 Cotes Chung
# MIT License
---
 
## <u>Vulnerability disclosures</u>

### [🛹 Critical vulnerability disclosed to lending protocol 88mph](#-critical-vulnerability-disclosed-to-lending-protocol-88mph)
I disclosed a critical bug to the fixed-rate lending protocol [88mph](https://88mph.app/) and helped rescue over $6.5m. Technical details around this vulnerability can be found at the posts by [iosiro](https://iosiro.com/blog/88mph-bug-bounty-post-mortem) and [Immunefi](https://medium.com/immunefi/88mph-function-initialization-bug-fix-postmortem-c3a2282894d3).

This bug was also eligble for the Founders Bounty, which included an $ARMOR token bonus, as well as the choice of my own tattoo on the [EaseDeFi](https://twitter.com/EaseDeFi) co-founder [Robert Forster](https://twitter.com/RobertMCForster). The tattoo was easily the most unique prize I've ever received, and can be [seen here!](https://twitter.com/RobertMCForster/status/1475869453556408325)


---
### [🧱 Critical vulnerability disclosed to Polygon](#-critical-vulnerability-disclosed-to-polygon)
I disclosed a denial-of-service bug to [Polygon](https://polygon.technology/) affecting their [`StakeManagerProxy`](https://etherscan.io/address/0x5e3ef299fddf15eaa0432e6e66473ace8c13d908) smart contract and its dependent contracts. Technical details around this vulnerability can be found at the post by [iosiro](https://iosiro.com/blog/temporary-denial-of-service-vulnerability-disclosed-to-and-remediated-by-polygon) and [Polygon](https://hackmd.io/SoItk4zvTDuJ2Rio5Byu_w).

---

### [🏌️ Critical vulnerability disclosed to four DeFi/NFT projects and escalated to OpenZeppelin](#️-critical-vulnerability-disclosed-to-four-definft-projects-and-escalated-to-openzeppelin)
I disclosed a critical bug involving to four DeFi/NFT projects, and prevented the following TVLs (>$50m) from permanent hack damage:

- KeeperDAO: ~$44m worth of tokens 
- Rivermen NFT: ~$6.95m worth of NFTs
- redacted project: <$2m worth of NFTs
- redacted project: <$500k worth of tokens

The root of the critical bug was an uninitalized logic contract behind a UUPS proxy leading to an arbitrary `delegatecall`, which could be used with a `selfdestruct` instruction. The severity was heightened since, by default, UUPS proxies have no external upgrade functions, so a `selfdestruct` call would permanently impair the proxy contract rendering all funds permently inaccessible. 

Technical details around this vulnerability can be found at the post by [iosiro](https://iosiro.com/blog/openzeppelin-uups-proxy-vulnerability-disclosure). The official OpenZeppelin postmortem be can be found [here](https://forum.openzeppelin.com/t/uupsupgradeable-vulnerability-post-mortem/15680).

---
### [🧑‍🎨 Critical vulnerability disclosed to @abwagmi](#-critical-vulnerability-disclosed-to-abwagmi)
I disclosed a critical bug to [@abwagmi](https://twitter.com/abwagmi/status/1465866170599358465) regarding a buggy `transferFrom` function on their AxonsToken contract that could allow any user to steal the entire circulating token supply. 

An overridden [`transferFrom` function in the `AxonsToken`](https://rinkeby.etherscan.io/address/0xd3cF1baab1F75d5bd86150963dda164c6E3E87A6#code#L687) allowed anyone to send tokens to the `auctionHouse` contract from any other account, or pull tokens from the `auctionHouse` contract without allowance. A malicious user could exploit this to send other user's tokens to the `auctionHouse` address and then pull it for themselves. The POC can be seen at the [following code sample](https://gist.github.com/AshiqAmien/470add84111539a724c35350dc30a49f).

Big thanks [@abwagmi](https://twitter.com/abwagmi/status/1466343883755995139) for supporting whitehats! 

---
### [⚡ Critical vulnerability disclosed to pxMythics](#-critical-vulnerability-disclosed-to-pxmythics)
I disclosed a critical bug to [pxMythics](https://twitter.com/pxmythicsnft/) regarding an access control vulnerability that could lead to permanent bricking of mint-related functions across two contracts.

The [Genesis](https://rinkeby.etherscan.io/address/0xa305F7078c8b2F9F95205e272aa680a86F003C34#code) and [GenesisSupply](https://rinkeby.etherscan.io/address/0x81360eDEF3b9F3639fA60639729881Aba9Fe29B1#code) contracts contained mint and mint-related functions such as the  [mintWhitelist](https://rinkeby.etherscan.io/address/0xa305F7078c8b2F9F95205e272aa680a86F003C34#code#F1#L141) function and the [airdrop](https://rinkeby.etherscan.io/address/0xa305F7078c8b2F9F95205e272aa680a86F003C34#code#F1#L120) function. These functions relied on the mint state being [closed or active](https://rinkeby.etherscan.io/address/0x81360eDEF3b9F3639fA60639729881Aba9Fe29B1#code#F1#L132), with the Genesis contract owner having [functionality to change the mint state](https://rinkeby.etherscan.io/address/0xa305F7078c8b2F9F95205e272aa680a86F003C34#code#F1#L87) as needed. However, both contracts contained an external unprotected [_setMintState](https://rinkeby.etherscan.io/address/0x81360eDEF3b9F3639fA60639729881Aba9Fe29B1#code#F3#L13) function that allowed anyone to update the mint state. Moreover, anyone could change the state to `Finalized`, which [bricks the aforementioned](https://rinkeby.etherscan.io/address/0xa305F7078c8b2F9F95205e272aa680a86F003C34#code#F1#L154) [mint-related functions](https://rinkeby.etherscan.io/address/0xa305F7078c8b2F9F95205e272aa680a86F003C34#code#F1#L127) and [prevents the mint state from being updated](https://rinkeby.etherscan.io/address/0x81360eDEF3b9F3639fA60639729881Aba9Fe29B1#code#F1#L124) further.

Many thanks to the [pxMythics](https://twitter.com/pxMythicsNFT/status/1480285214140162053) team for supporting whitehats! 

---
### [🧪 High risk vulnerability disclosed to Alchemix](#-high-risk-vulnerability-disclosed-to-alchemix)
I disclosed an access-control bug to future yield tokenization protocol [Alchemix](https://alchemix.fi/). Technical details around this vulnerability can be found at the post by [iosiro](https://iosiro.com/blog/high-risk-vulnerability-disclosed-to-alchemix) and [Immunefi](https://medium.com/immunefi/alchemix-access-control-bug-fix-debrief-a13d39b9f2e0).

---
### [🌀 High risk vulnerability disclosed to Ondo Finance](#-high-risk-vulnerability-disclosed-to-ondo-finance)
I disclosed a potential token theft bug to DeFi protocol [Ondo Finance](https://ondo.finance/). The exploit relied on a feature that was not live, so no actual user funds were at risk. Technical details around this vulnerability can be found at the post by [iosiro](https://iosiro.com/blog/high-risk-vulnerability-disclosed-to-ondo-finance).


---
### [🕵️ Undisclosed vulnerabilities](#-undisclosed-vulnerabilities)

I've reported bugs that were confirmed by the respective team, but they've asked to redact the bug details in the interim. If this changes in the future, I'll update the list below and include the details once I have permission.

- Medium risk bug reported to and confirmed by *redacted project 1*.
- High risk bug reported to and confirmed by *redacted project 2*.
- High risk bug reported to and confirmed by *redacted project 3*.
- Low risk bug reported to and confirmed by *redacted project 4*.
- Critical bug reported to and confirmed by *redacted project 5*.

---
### [🥊 Code 4rena contests](#-code-4rena-contests)
I occassionally participate in [code 4rena](https://code423n4.com/) contests, usually quite loosely. You can find my progress on the [leaderboard](https://code423n4.com/leaderboard) under the alias `toastedsteaksandwich`. I've submitted at least one valid issue to the following contests:

- [Wild Credit](https://code423n4.com/reports/2021-07-wildcredit/)
- [Based Loans](https://code423n4.com/reports/2021-04-basedloans/)
- [Vader Protocol](https://code423n4.com/reports/2021-04-vader/)
- [Visor Finance](https://code423n4.com/reports/2021-05-visorfinance/)
- [Streaming Protocol](https://code4rena.com/reports/2021-11-streaming)

---


## <u>Training materials</u>

### [✍️ Introduction to smart contract bug hunting](#-introduction-to-smart-contract-bug-hunting)
I wrote a blog post for [Hack South](https://hacksouth.africa/) to introduce smart contract bug hunting. The post can be found [here](https://hacksouth.africa/bug%20bounty/smart-contract-bug-hunting/). 

---
### [🧑‍🏫 How to PoC your bug leads](#-how-to-poc-your-bug-leads)

I wrote a blog post for [Immunefi](https://immunefi.com/) on how to write a PoC for any smart contract bug leads you've come across. The post can be found [here](https://medium.com/immunefi/how-to-poc-your-bug-leads-5ec76abdc1d8). 

---
### [🤸‍♀ Getting started with smart contract bug bounty](#-getting-started-with-smart-contract-bug-bounty)

I wrote a blog post for [YesWeHack](https://www.yeswehack.com/) on getting started with smart contract bug bounty. The post can be found [here](https://blog.yeswehack.com/yeswerhackers/getting-started-smart-contract-bug-bounty/). 

---
### [🗣 BSides Cape Town 2022 Presentation](#-bsides-cape-town-2022-presentation)

I spoke at [BSides Cape Town 2022](https://twitter.com/BSidesCapeTown/status/1599019367856906241?cxt=HHwWgoCloZTp7bAsAAAA) about bricking smart contract proxies. The live recording can be found [here](https://www.youtube.com/live/fQBh_7i8R84?feature=share&t=16342).

---

## <u>Other</u>

### [😇 2022 Underhanded Solidity Contest](#-2022-underhanded-solidity-contest)

I entered the [2022 Underhanded Solidity Competition](https://underhanded.soliditylang.org/). My submission was based on Dedaub's vulnerability disclosure around [phantom functions](https://media.dedaub.com/phantom-functions-and-the-billion-dollar-no-op-c56f062ae49f). My final submission can be found [here](https://github.com/ethereum/solidity-underhanded-contest/tree/master/2022/submissions_2022/submission2_Ashiq).

---
### [🤓 yAcademy Block III](#-yacademy-block-iii)

I was selected to participate in [Block III](https://medium.com/yacademyblog/blocks-ii-iii-retrospective-2badd879b70e) of [yAcademy](https://yacademy.dev/). I worked along side other fellows to discuss and report bugs affecting in-scope codebases.
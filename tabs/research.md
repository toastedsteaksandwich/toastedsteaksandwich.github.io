---
title: Research and Training

# The Research page
# v2.0
# https://github.com/cotes2020/jekyll-theme-chirpy
# © 2017-2019 Cotes Chung
# MIT License
---
 
## <u>Vulnerability disclosures</u>

### 🛹 Critical vulnerability disclosed to lending protocol 88mph
I disclosed a critical bug to the fixed-rate lending protocol [88mph](https://88mph.app/) and helped rescue over $6.5m. Technical details around this vulnerability can be found at the posts by [iosiro](https://iosiro.com/blog/88mph-bug-bounty-post-mortem) and [Immunefi](https://medium.com/immunefi/88mph-function-initialization-bug-fix-postmortem-c3a2282894d3).


---
### 🧱 Critical vulnerability disclosed to Polygon
I disclosed a denial-of-service bug to [Polygon](https://polygon.technology/) affecting their [`StakeManagerProxy`](https://etherscan.io/address/0x5e3ef299fddf15eaa0432e6e66473ace8c13d908) smart contract and its dependent contracts. Technical details around this vulnerability can be found at the post by [iosiro](https://iosiro.com/blog/temporary-denial-of-service-vulnerability-disclosed-to-and-remediated-by-polygon) and [Polygon](https://hackmd.io/SoItk4zvTDuJ2Rio5Byu_w).

---

### 🏌️ Critical vulnerability disclosed to four DeFi/NFT projects and escalated to OpenZeppelin
I disclosed a critical bug involving to four DeFi/NFT projects, and prevented the following TVLs (>$50m) from permanent hack damage:

- KeeperDAO: ~$44m worth of tokens 
- Rivermen NFT: ~$6.95m worth of NFTs
- redacted project: <$2m worth of NFTs
- redacted project: <$500k worth of tokens

The root of the critical bug was an uninitalized logic contract behind a UUPS proxy leading to an arbitrary `delegatecall`, which could be used with a `selfdestruct` instruction. The severity was heightened since, by default, UUPS proxies have no external upgrade functions, so a `selfdestruct` call would permanently impair the proxy contract rendering all funds permently inaccessible. 

Technical details around this vulnerability can be found at the post by [iosiro](https://iosiro.com/blog/openzeppelin-uups-proxy-vulnerability-disclosure). The official OpenZeppelin postmortem be can be found [here](https://forum.openzeppelin.com/t/uupsupgradeable-vulnerability-post-mortem/15680).

---
### 🧑‍🎨 Critical vulnerability disclosed to [@abwagmi](https://twitter.com/abwagmi/)
I disclosed a critical bug to [@abwagmi](https://twitter.com/abwagmi/status/1465866170599358465) regarding a buggy `transferFrom` function on their AxonsToken contract that could allow any user to steal the entire circulating token supply. 

An overridden [`transferFrom` function in the `AxonsToken`](https://rinkeby.etherscan.io/address/0xd3cF1baab1F75d5bd86150963dda164c6E3E87A6#code#L687) allowed anyone to send tokens to the `auctionHouse` contract from any other account, or pull tokens from the `auctionHouse` contract without allowance. A malicious user could exploit this to send other user's tokens to the `auctionHouse` address and then pull it for themselves. The POC can be seen at the [following code sample](https://gist.github.com/AshiqAmien/470add84111539a724c35350dc30a49f).

Big thanks [@abwagmi](https://twitter.com/abwagmi/status/1466343883755995139) for supporting whitehats! 

---
### 🧪 High risk vulnerability disclosed to DeFi protocol Alchemix
I disclosed an access-control bug to The future yield tokenization protocol [Alchemix](https://alchemix.fi/). Technical details around this vulnerability can be found at the post by [iosiro](https://iosiro.com/blog/high-risk-vulnerability-disclosed-to-alchemix) and [Immunefi](https://medium.com/immunefi/alchemix-access-control-bug-fix-debrief-a13d39b9f2e0).

---
### 🕵️ Undisclosed vulnerabilities

I've reported bugs that were confirmed by the respective team, but they've asked to redact the bug details in the interim. If this changes in the future, I'll update the list below and include the details once I have permission.

- Medium risk bug reported to and confirmed by *redacted project 1*.
- High risk bug reported to and confirmed by *redacted project 2*.
- High risk bug reported to and confirmed by *redacted project 3*.
- Low risk bug reported to and confirmed bt *redacted project 4*.

---
### 🥊 Code 4rena contests
I occassionally participate in [code 4rena](https://code423n4.com/) contests, usually quite loosely. You can find my progress on the [leaderboard](https://code423n4.com/leaderboard) under the alias `toastedsteaksandwich`. I've submitted at least one valid issue to the following contests:

- [Wild Credit](https://code423n4.com/reports/2021-07-wildcredit/)
- [Based Loans](https://code423n4.com/reports/2021-04-basedloans/)
- [Vader Protocol](https://code423n4.com/reports/2021-04-vader/)
- [Visor Finance](https://code423n4.com/reports/2021-05-visorfinance/)

---


## <u>Training materials</u>

### ✍️ Introduction to smart contract bug hunting
I wrote a blog post for [Hack South](https://hacksouth.africa/) to introduce smart contract bug hunting. The post can be found [here](https://hacksouth.africa/bug%20bounty/smart-contract-bug-hunting/). 

---
### 🧑‍🏫 How to PoC your bug leads

I wrote a blog post for [Immunefi](https://immunefi.com/) on how to write a PoC for any smart contract bug leads you've come across. The post can be found [here](https://medium.com/immunefi/how-to-poc-your-bug-leads-5ec76abdc1d8). 

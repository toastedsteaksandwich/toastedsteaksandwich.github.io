---
title: Research

# The Research page
# v2.0
# https://github.com/cotes2020/jekyll-theme-chirpy
# © 2017-2019 Cotes Chung
# MIT License
---

This page documents some of the research work I've done, as well as some of the training content I've provided. Further information around most disclosed bugs detailed here can be found in it's summary. 


---
## 🛹 Critical vulnerability disclosed to lending protocol 88mph
I disclosed a critical bug to the fixed-rate lending protocol [88mph](https://88mph.app/) and helped rescue over $6.5m. Technical details around this vulnerability can be found at the posts by [iosiro](https://iosiro.com/blog/88mph-bug-bounty-post-mortem) and [Immunefi](https://medium.com/immunefi/88mph-function-initialization-bug-fix-postmortem-c3a2282894d3).

---
## 🧪 High risk vulnerability disclosed to DeFi protocol Alchemix
I disclosed an access-control bug to The future yield tokenization protocol [Alchemix](https://alchemix.fi/). Technical details around this vulnerability can be found at the post by [iosiro](https://iosiro.com/blog/high-risk-vulnerability-disclosed-to-alchemix) and [Immunefi](https://medium.com/immunefi/alchemix-access-control-bug-fix-debrief-a13d39b9f2e0).

---
## 🧱 Critical vulnerability disclosed to Polygon
I disclosed a denial-of-service bug to [Polygon](https://polygon.technology/) affecting their [`StakeManagerProxy`](https://etherscan.io/address/0x5e3ef299fddf15eaa0432e6e66473ace8c13d908) smart contract and its dependent contracts. Technical details around this vulnerability can be found at the post by [iosiro](https://iosiro.com/blog/temporary-denial-of-service-vulnerability-disclosed-to-and-remediated-by-polygon) and [Polygon](https://hackmd.io/SoItk4zvTDuJ2Rio5Byu_w).

---
## 🏌️ Critical vulnerability disclosed to four DeFi/NFT projects and escalated to OpenZeppelin
I disclosed a critical bug involving to four DeFi/NFT projects, and prevented the following TVLs (>$50m) from permanent hack damage:

- KeeperDAO: ~$44m worth of tokens 
- Rivermen NFT: ~$6.95m worth of NFTs
- redacted project: <$2m worth of NFTs
- redacted project: <$500k worth of tokens

The root of the critical bug was an uninitalized logic contract behing UUPS proxy leading to an arbitrary `delegatecall`, which could be used with a `selfdestruct` instruction. The severity was heightened since, by default, UUPS proxies have no external upgrade functions, so a `selfdestruct` call would permanently impair the proxy contract rendering all funds permently inaccessible. 

Technical details around this vulnerability can be found at the post by [iosiro](https://iosiro.com/blog/openzeppelin-uups-proxy-vulnerability-disclosure). The official OpenZeppelin postmortem be can be found [here](https://forum.openzeppelin.com/t/uupsupgradeable-vulnerability-post-mortem/15680).

---
## 🕵️ Undisclosed vulnerabilities

I've reported bugs that were confirmed by the respective team, but they've asked to redact the bug details in the interim. If this changes in the future, I'll update the list below and include the details once I have permission.

- Medium risk bug reported to and confirmed by *redacted project 1*.
- High risk bug reported to and confirmed by *redacted project 2*.
- High risk bug reported to and confirmed by *redacted project 3*.

---
## 🥊 Code 4rena contests
I occassionally participate in [code 4rena](https://code423n4.com/) contests, usually quite loosely. You can find my progress on the [leaderboard](https://code423n4.com/leaderboard) under the alias `toastedsteaksandwich`. I've submitted at least one valid issue to the following contests:

- [Wild Credit](https://code423n4.com/reports/2021-07-wildcredit/)
- [Based Loans](https://code423n4.com/reports/2021-04-basedloans/)
- [Vader Protocol](https://code423n4.com/reports/2021-04-vader/)
- [Visor Finance](https://code423n4.com/reports/2021-05-visorfinance/)

---
## ✍️ Introduction to smart contract bug hunting
I wrote a blog post for [Hack South](https://hacksouth.africa/) to introduce smart contract bug hunting. The post can be found [here](https://hacksouth.africa/bug%20bounty/smart-contract-bug-hunting/). 

---
## 🧑‍🏫 How to PoC your bug leads

I wrote a blog post for [Immunefi](https://immunefi.com/) on how to write a PoC for any smart contract bug leads you've come across. The post can be found [here](https://medium.com/immunefi/how-to-poc-your-bug-leads-5ec76abdc1d8). 

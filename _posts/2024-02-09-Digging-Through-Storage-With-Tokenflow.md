---
title: ⛏️ Digging through contract storage with TokenFlow 
author: Ashiq Amien
date: 2024-02-08 17:00:00 +0200
categories: [Storage, Research]
tags: Research
math: true
---

# Intro

This blog post is written to demonstrate [TokenFlow's](https://tokenflow.live/) [Blockchain Datasets](https://tokenflow.live/products/blockchain-datasets/). I used the Ethereum and Optimism datasets to query historical storage of certain contracts during bug hunting and just odds-and-ends research. The info I'll be presenting here are missed bugs, or otherwise just interesting artefacts that I found on-chain.

## Optimism's second `TransferOnion`

Last year I got an OP token airdrop from Optimism, and instead of needing to claim it with a merkle proof, the tokens landed up in my wallet without any interaction. I dug around to see how they did this, since I figured they wouldn't be manually transferring 1000s of tokens. The airdrop was facilitated with something they called a [`TransferOnion`](https://optimistic.etherscan.io/address/0x53466d3cabb3b59aa6065fd732672d68f191efc0#code). It works as follows:

1) A contract is deployed with a token address, a token sender address and a 32-byte hash (called a shell). 
2) A `peel()` function is called, and takes in a `layer`, consisting of a recipient, an amount, and a corresponding `shell`. If the shell set from the constructor matches `keccak256(abi.encode(layer.recipient, layer.amount, layer.shell))`, the OP tokens are transferred from the sender to the recipient.
3) The contract-wide `shell` is updated to the shell from the current layer that's being `peel`'ed. 
4) Repeat for all layers until the shell is set to 0x0.

The `peel()` function is permissionless, so anyone could start the token distribution with the correct layer information. Of course, this relies on the token sender setting an approval to this `TransferOnion` contract for the OP tokens to be transferred.

I noticed that the contract deployer deployed two `TransferOnions` but [one of them was unpeeled](0x2bEF9b5f647C0f36C082365E06d68CEf5331eF8F). Moreover, the shell was set to the same original shell as the contract that was peeled. 

![A ripe on-chain onion](/assets/img/sample/tokenflow-onion.png)

Immediately, since the shell was the same, I thought there could be a way to double on everyone's airdrop by peeling the `TranferOnion` with the exact same layers. However, there was no allowance set from the spender, so the tx would just revert. But I wanted to know- was there any time that the spender accidentally approved this contract, and then revoked the allowance at a later stage? This is where I used TokenFlow's Optimism data set to query the historical storage of the OP token contract to see if the spender had done this before. The dataset is accessible through [Snowflake](https://www.snowflake.com/en/) and SQL is used to construct queries. The query to check the OP token's storage history of the spender's allowance is as follows:

```sql
SELECT 
    curr_value,
    tx_hash,
    location,        
    prev_value,
    block_number    
 FROM OPTIMISM.storage_diffs
 WHERE 
    contract_address = LOWER('0x4200000000000000000000000000000000000042')
 AND location LIKE LOWER('%allow%0x2501c477D0A35545a387Aa4A3EEe4292A9a8B3F0%0x2bEF9b5f647C0f36C082365E06d68CEf5331eF8F')
```

Here, I'm checking the location history of 0x2501...B3F0 (the token sender) to see if any allowance was granted to 0x2bEF...eF8F (the second onion). The syntax to match this exact location can be done in several ways, but just to simplify things for all my queries, I use `%allow%` to match the allowance location. `%` is a wildcard character that can be substituted for one or more characters, and since the allowance location can be labelled as different things (e.g. `allowed`, `_allowance`, `allowances`, ...) the wildcard covers for these potential matches so I don't need to check the labels for other tokens. Then, `TokenFlow` labels storage mappings with square brackets, but just out of laziness, `%` works well enough to separate the addresses in the query. 

Trying to dig historical contract storage can be a pain. As a comparison, you could try to fork the chain and query for historical allowances. This means you'd need to iterate over many many blocks (over 6 million blocks) in hopes to find something. Of course, a more feasable alternative to the above would be to check for event emissions from the token contract. However, not all contracts emit events for important state changes, which makes storage digging more difficult.

## Digging out Degenbox's V4 cauldrons

Last year, [Abracadabra Money released a post mortem](https://mirror.xyz/0x5744b051845B62D6f5B6Db095cc428bCbBBAc6F9/47LK6nUpMrVsYzfCYBTyZsc_7t5Sh5onxO8sSEotNMY) indicating that their V4 cauldrons were vulnerable and that they white-hacked the funds to protect their users. I wanted to dig into the mechanics of this bug, but never got round to it. With my access to TokenFlow, I figured it'll be a case of running a few queries and I'll be able to dig out everything that happened. 

### Degenbox access control

Just as a quick refresher, it's worth talking about how Degenbox interacts with `masterContracts`, and how it manages access control. A `masterContract` is a contract template that sits on top of Degenbox that can manage user funds. `MasterContract` instances are permissionlessly deployed and registered on Degenbox. Users can authorize a `masterContract` to spend their funds by submitting an explicit signature, or otherwise just perform a `setMasterContractApproval()` call if the `masterContract` was whitelisted by the owner. It's noteworthy that the user authorizes the `masterContract` only, so any instance of the same `masterContract` has the same permission as any other instance.

### V4 attack vectors

The V4 cauldrons were vulnerable because there was a small window where an attacker could exploit the cauldron to perform an arbitrary call to itself, or back to the Degenbox. If a user approved one of these vulnerable cauldrons, then there's at least 2 main ways their funds could be stolen:

1) Any Degenbox balances could be stolen (arbitrary tokens, not just Cauldron collaterals)
2) Any token balances on the account could be stolen, provided an allowance was provided to Degenbox

To see if any of the above is a viable path, we first need to check which users approved either of the buggy cauldrons. The 2 main buggy Cauldron `masterContracts` are 0x43243F7BdDCb850acB687c42BBf5066c224054a5 and 0xb2EBF227188E44ac268565C73e0fCd82D4Bfb1E3. It's easy to check who still has approval to either of these cauldrons with the following query:

```sql
SELECT 
    tx_hash,
    location,        
    curr_value,
    prev_value,
    block_number    
FROM MARKETPLACE.storage_diffs
WHERE 
    (
    contract_address = LOWER('0xd96f48665a1410C0cd669A88898ecA36B9Fc2cce') //degenbox
    )   
AND location like ('%masterContractApproved[0xb2ebf227188e44ac268565c73e0fcd82d4bfb1e3%')

LIMIT 50
```

![Approvals](/assets/img/sample/tokenflow-approvals.png)

We can sift the results to find the users who are still approved to `0xb2eb...b1e3`:

```
0xe0d6b751ab1b28098d581d1f2265e76e16a3f10e
0x21b3edd21f609c5f68d6ede16bf66dc890c56b7b
0x4edaf9d6b34b1eb3ca712054ded4c87af9c4969b
0x6f36212aad5ffb626f09f1a32ade7c13b3cc389e
0xa84842f23737367274b6949e61f0ba8f3239b0bf
0xeefb8e56264ac1b3f7207dc5d7154d20a22b9359
```

To find the token balances, we can go through the storage of each address individually and collect the latest storage update per token, but this seems a bit tedious. So let's just grab all tokens associated any of the above addresses and then run through the latest Degenbox balance to see if there's anything avail:

```sql
SELECT 
    tx_hash,
    location,        
    curr_value,
    prev_value,
    block_number    
FROM MARKETPLACE.storage_diffs
WHERE 
    (
    contract_address = LOWER('0xd96f48665a1410C0cd669A88898ecA36B9Fc2cce')
    )   
AND location like '%balanceOf[%][0xe0d6b751ab1b28098d581d1f2265e76e16a3f10e]'
...
OR location like '%balanceOf[%][0xeefb8e56264ac1b3f7207dc5d7154d20a22b9359]'

```

We can sort the unique token addresses from the output to give us:

```
0xca76543cf381ebbb277be79574059e32108e3e65
0x26fa3fffb6efe8c1e69103acb4044c26b9a106a9
0x99d8a9c45b2eca8864373a26d1459e3dff1e17f3
0xdcd90c7f6324cfa40d7169ef80b12031770b4325
0xc02aaa39b223fe8d0a0e5c4f27ead9083c756cc2
0x6b175474e89094c44da98b954eedeac495271d0f
0x8798249c2e607446efb7ad49ec89dd1865ff4272
0xa0b86991c6218b36c1d19d4a2e9eb0ce3606eb48
0xdf0770df86a8034b3efef0a1bb3c889b8332ff56
0xdac17f958d2ee523a2206206994597c13d831ec7
```

Now, there's two checks to perform. One is to see if there's any balance on Degenbox using the `balanceOf()` method, and the second is if there's any allowance to Degenbox with the above tokens. It's noteworthy that the user may have approved a cauldron's collateral token directly to Degenbox to used with the [`_bentoDeposit()`](https://etherscan.io/address/0x207763511da879a900973a5e092382117c3c1588#code#F8#L380) function, so those should be added too if they weren't picked up. To check the Degenbox balances, anything similar to the following code will work:

```bash
#!/bin/bash

# First list of addresses for var1
addresses_var1=(
    "0xca76543cf381ebbb277be79574059e32108e3e65"
     ...
    "0xdac17f958d2ee523a2206206994597c13d831ec7"
    "0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599" # cauldron collateral
)

# Second list of addresses for var2
addresses_var2=(
    "0xe0d6b751ab1b28098d581d1f2265e76e16a3f10e"
    ...
    "0xeefb8e56264ac1b3f7207dc5d7154d20a22b9359"
)

# Iterate over each address in var1
for var1 in "${addresses_var1[@]}"; do
    # Iterate over each address in var2
    for var2 in "${addresses_var2[@]}"; do
        # Run the cast call command with the current pair of addresses
        cast call 0xd96f48665a1410C0cd669A88898ecA36B9Fc2cce "balanceOf(address,address)(uint256)" $var1 $var2 --rpc-url YOUR_RPC_HERE
    done
done

```

You could probably do above using multicall or something more efficient, but it's a small set and works well enough. To check for allowances to approved to Degenbox, we can just switch out the following line:

```bash
cast call $var1 "allowance(address,address)(uint256)" $var2 0xd96f48665a1410C0cd669A88898ecA36B9Fc2cce --rpc-url YOUR_RPC_HERE
```

Luckily, everything comes out clean, or at least negligble after gas. And we can repeat for the process for 0x43243F7BdDCb850acB687c42BBf5066c224054a5 to find a similar result :)


### Cross contract dud

Degenbox is a fork of Bentobox from Sushiswap, with very similar code (if not the same?) and similar architecture. While doing the above checks I had the idea that cross contract access control might be an issue if the same cauldron or kashi `masterContract` is whitelisted on both Degenbox and Bentobox. This would be an issue since the `masterContracts` could abuse the arbitrary [`call()`](https://etherscan.io/address/0x43243f7bddcb850acb687c42bbf5066c224054a5#code#F8#L405) method on the other box, as the corresponding box wouldn't be blacklisted. This could be exploited since the `masterContracts` could call the [`registerProtocol()`](https://etherscan.io/address/0xF5BCE5077908a1b7370B9ae04AdC565EBd643966#code#L520) and then the [access control would be bypassed](https://etherscan.io/address/0xF5BCE5077908a1b7370B9ae04AdC565EBd643966#code#L749) for any user who approved the doubled `masterContract`. To go about this, I dumped all of the unique `masterContracts` with the following query:

```sql
SELECT 
    curr_value
FROM 
    (SELECT DISTINCT
        curr_value,
        tx_hash,
        location,        
        prev_value,
        block_number    
     FROM MARKETPLACE.storage_diffs
     WHERE 
        contract_address = LOWER('0xF5BCE5077908a1b7370B9ae04AdC565EBd643966') --bentobox
        AND location LIKE '%masterContractOf%'
    ) AS unique_values
GROUP BY 
    curr_value

```

Notice that I'm not just dumping whitelisted contracts- that's because a user could send a signature and approve any contract, so I used `%masterContractOf%` for completeness sake. I did the same for Degenbox and found that there were no overlap in contracts, which was enough for me to conclude that this was a non-issue. (Of course, if a user explicitly approved any cauldron/kashi that belonged to the other box, this would be an issue. But this seems unlikely for any user to do by accident).

After all this effort, I realized I completely overlooked a simple fact. All cauldrons and kashi contracts store the box they're building on top of in the constructor! So this scenario was already very unlikely to be an issue at all. But in any case, using TokenFlow made this check fairly simple to validate.

----
Many thanks to TokenFlow for giving me access to their Ethereum and Optimism Datasets, I had a lot of fun exploring these contracts! Be sure to check them out [here](https://tokenflow.live/). (This was not paid/sponsored etc.)


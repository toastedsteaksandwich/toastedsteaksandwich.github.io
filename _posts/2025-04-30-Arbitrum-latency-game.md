---
title: 🏃💨 Playing Arbitrum's latency games for fun and profit
author: Ashiq Amien
date: 2025-04-30 17:15:00 +0200
categories: [Research, MEV]
tags: research
---

## Intro

A few days ago, Arbitrum [introduced a new transaction ordering policy called Timeboost](https://x.com/arbitrum/status/1912945720237367449). Before Timeboost, transactions were ordered first-come-first-serve (FCFS), leading to searchers racing to minimize latency by fractions of a second to land their tx first to capture whatever MEV they've detected. This blog post details my experience in attempting to capture (long-tail) MEV and participating in this latency game. 

### Background

Compared to Ethereum, Arbitrum has a centralized sequencer that batches incoming transactions according to the FCFS policy. This means a few important things:

- There's no mempool, so there's no generalized (or any) front-running
- Your gas price doesn't impact your position in a block, it just needs to suffice for your tx
- Transactions are ordered by the [Sequencer](https://docs.arbitrum.io/how-arbitrum-works/transaction-lifecycle), so the faster you reach the Sequencer, the quicker your tx will be included.

So what's the game to play if there's no front-running? All MEV captures will need to take place as a back-run, which is considered less harmful than any front-running since there's no sandwiching, harmful JIT transactions, etc. Once an opportunity is available (i.e. some arb, liquidation, long-tail opp, etc) it becomes a race between competing searchers to land their tx first, since typically it takes one tx to capitalize on the opportunity.  

This race causes searchers to dissect the end-to-end process from monitoring for opportunities all the way to getting your tx to backrun the opportunity as close as possible. Of course, if you're not a 31337-dank-super-coder then you'll need to start somewhere that's a bit closer to your current skillset, like I did. Let's dive into it 👇 

### The naive approach

I stumbled on a simple long-tail MEV opportunity that leaked some value after large trades were taking place. After evaluating things (and making sure this is not a bug!), I saw you could backrun these trades and `rebalance` the pool and net the imbalance for yourself. I knew there were other searchers already monitoring this opportunity, but it seemed quiet enough to give it a try. 

Not knowing the full extent of the latency games on Arbitrum, I just went with the naive approach: script the monitoring and tx submission in your favourite language, use some 3rd party RPC to read and write to the chain, and toss it somewhere to run. These were some of the specs I ended up with: 

- I used a paid Chainstack node as my 3rd party RPC 
- I used node.js to write my script and hosted it on an EC2 (Cape Town)
- I was somewhat rate limited, so I polled for opportunities every ~10 blocks  
- On each poll I queried multiple different contracts and tokens
- I generated a tx to capture the MEV every time the opportunity was available, tailored with parameters for the opportunity itself
- I queried for the gas price before tx submission
- Finally, I submitted my tx to my Chainstack node 


Surprisingly, this approach worked for a few days. However, it wasn't long before a competitor showed up, and very soon I was outpaced to land a tx. I did a few more optimizations as follows:

- I upgraded my Chainstack node and polled every block
- I used multicall to read all the necessary state I needed in a single call to prove that there is an opportunity to capture
- I deployed a contract that I could easily call to capture the opportunity with a single parameter, instead of generating complex txs locally
- I moved my EC2 around to see what would land faster
- I dropped gas price queries because Arbitrum gas was cheap enough and I could overpay slightly

With these updates I was in the lead again, but I still wasn't backrunning within 1 or 2 blocks, rather just a couple blocks. To stay ahead, I knew I had to race much faster and that's when I found a super helpful alpha leak on Twitter to help with that 🔐
 


### Diving into alpha leaks

I stumbled on [this thread by duoxehyon](https://x.com/duoxehyon/status/1707366729574429081) and I was surprised to see it spelled out pretty much the exact steps on how to significantly improve your latency race. This thread explained what it means to read from the Sequencer, where to co-locate to the Sequencer and even how to do keep-alive on connections. 

I had a new direction: this meant re-architecting the bot to read the sequencer feed directly from the sequencer, and writing to it as fast as possible. Arbitrum has docs on how to read the [sequencer feed](https://docs.arbitrum.io/run-arbitrum-node/sequencer/read-sequencer-feed), but duoxehyon [wrote a client](https://github.com/duoxehyon/sequencer-client-rs) to do this in Rust, so I used that instead of building my own since Rust is known to have great performance. 

I should note that by the time I found the Rust sequencer-client, I had *zero* experience with Rust and couldn't get to read anything from the feed out of the box. Moreover, once I could get it running, I noticed the feed was not printing nearly as many transactions as actually landed on-chain. Slowly but surely, by piecing together info from the [docs](https://docs.arbitrum.io/run-arbitrum-node/sequencer/read-sequencer-feed), with sufficient print statements to trace and help from Claude/ChatGPT, I landed on being able to read all txs as they were being sequenced:

```javascript
loop {        
    let data = receiver
        .recv()
        .expect("Failed to receive data from feed client");        

    let msg_seq_num = data.messages[0].sequence_number;
    let l2msg = data.messages[0].message.message.l2msg.clone();
    let kind = data.messages[0].message.message.header.kind.clone();      

    if highest_seq_number >= msg_seq_num {
        continue;
    }

    highest_seq_number = msg_seq_num;        

    let decoded = match decode(l2msg) {
        Ok(res) => res,
        Err(err) => {
            eprintln!("Error decoding base64: {}", err);
            return;
        }
    };

    //L1_MESSAGE_TYPE_L2_MESSAGE == 3
    if(kind == 3){
        let l2kind = &decoded[..2];
        
        if(l2kind[0] == 0){info!("type 0 l2msg: "); }
        if(l2kind[0] == 1){info!("type 1 l2msg: "); }            
        //if(l2kind[0] == 3){info!("type 3 l2msg: "); }

        if(l2kind[0] == 4){
            
            if(l2kind[1] > 0x7f ){
                //legacy tx
                let rlp = Rlp::new(&decoded[1..]);//slice the first byte off
                let decoded_tx: Transaction = rlp.as_val().expect("Failed to decode transaction");
                info!("l2msg: {:?} ", decoded_tx.to);                  
            }

            else if(l2kind[1] == 1){
                let rlp = Rlp::new(&decoded[1..]);//slice the first byte off
                let decoded_tx: Transaction = rlp.as_val().expect("Failed to decode transaction");                    
            }
            else if(l2kind[1] == 2){
                //it's an eip 1559 tx
                let rlp = Rlp::new(&decoded[1..]);
                let decoded_tx: TypedTransaction = rlp.as_val().expect("Failed to decode transaction");
                info!("EIP-1559 Transaction: {:?}", decoded_tx.to());
            }
            else{
                info!("idk man");
            }
        }
    }   
}
```
Now that txs were being read, I had to figure out which txs to backrun. Since I didn't want to involve poll RPC on each block to keep my performance high, I did some investigations and found the resulting txs that open the MEV opportunity. From here, it was a matter of reading the feed and matching the calldata + tx sender to some hard coded values:

```javascript
let decoded = data.messages[0].message.message.decode();        
        
if let Some(DecodedMsg::DecodedBatch(ref transactions)) = decoded {
    for transaction in transactions {                
        let incoming_sig = extract_first_4_bytes(&transaction.input);
        if(incoming_sig == first_target_bytes || incoming_sig == second_target_bytes) {
            // This ensures that `transaction.to` is `Some` and handle the `None` case
            if let Some(to_address) = transaction.to {
                //Check if the transaction's `to` address matches any target address from investigation 
                //This fetches the associated address that we need to backrun from a hardcoded mapping
                if let Some(associated_address) = target_map.get(&to_address) {                                      
                    match tx_sender.send(value, *associated_address).await {
                        Ok(response) => {
                            if let Some(error) = response.error {
                                // Handle error response
                                println!("Transaction failed with error: {:?}", error);
                            } else if let Some(result) = response.result {                                
                                //println!("Transaction succeeded with result: {:?}", result);                                
                            } else {
                                println!("Unexpected response format: {:?}", response);
                            }
                        },
                        Err(e) => eprintln!("Error sending transaction: {:?}", e),
                    }
                }   
        }
     }
}
```

Once a heuristic tx is identified, we need to send in our backrun tx to the sequencer as fast as possible. Again, with a fair dose of trial-and-error and help from AI chatbots, I managed to piece together the following:

```javascript
async fn send(&self, value: U256, market_address: Address) -> Result<SequencerResponse, Box<dyn Error>> {
    let mut nonce = self.nonce.lock().unwrap(); // Lock and access nonce    
    let func_signature = "backrun(address)".as_bytes();
    let function_selector = &keccak256(func_signature)[..4];

    // Encode the parameter and make sure that the bytes are 4 + 32 at least
    let mut encoded_address = [0u8; 32];
    encoded_address[12..].copy_from_slice(&market_address.as_bytes());

    // Combine function selector and encoded parameter
    let mut data = Vec::with_capacity(4 + 32); // 4 bytes for function selector + 32 bytes for address
    data.extend_from_slice(function_selector);
    data.extend_from_slice(&encoded_address);

    let tx = Eip1559TransactionRequest::new()        
    .to("0xaddress".parse::<Address>()?)
    .value(value)
    .data(Bytes::from(data))
    .chain_id(self.chain_id) // Chain ID for Arbitrum
    .gas(U256::from(900_000u64))            
    .max_fee_per_gas(U256::from(1_000_000_000u64)) 
    .max_priority_fee_per_gas(U256::from(1_000_000_000u64)) 
    .nonce(*nonce); // Nonce
                
    let signature = self.wallet.sign_transaction(&tx.clone().into()).await?;        
    let signed_tx = tx.rlp_signed(&signature);        
    let signed_tx_hex = hex::encode(signed_tx);    
    
    let response = self.client.post("https://arb1-sequencer.arbitrum.io/rpc")
        .header("accept", "application/json")
        .header("content-type", "application/json")
        .json(&json!({
            "id": 1,
            "jsonrpc": "2.0",
            "method": "eth_sendRawTransaction",
            "params": [format!("0x02{}", signed_tx_hex)]//need the 02 because it's an EIP1559 tx...
        }))
        .send()
        .await?
        .json::<SequencerResponse>()
        .await?;
```

After compiling and deploying a release version on a [Hetzner](https://x.com/duoxehyon/status/1707366750529196144) server instead of an EC2 instance, I was finally able to get some single-block backruns! 🏃 It was not perfect of course, there were still some issues and improvements:

- It wasn't always single-block backruns, it was sometimes two blocks behind, but always ahead of the competition at least. I did not implement this [alpha leak](https://x.com/duoxehyon/status/1707366779033674143), but perhaps if I did it would have been more consistent.
- Experienced Rust devs will likely be able to pick out several errors from the code I've shared. I generally never code for flair; I almost always prioritize functionality over polish, which served my aim. I should probably pick up Rust more formally at some point since the performance really was much better than my previous bots.
- Occasionally I wouldn't land a profitable tx, either because of weird gas price issues or because the heuristic on the addresses to backrun didn't actually provide much value to scoop up. This in general was okay because gas costs were so negligible. 

Overall, I was really happy that my frankenstein code worked. It outpaced the competition for a fair amount of time. At some point they simply spammed txs in hopes to randomly land a backrun before I did (again, because gas is so cheap). Eventually they managed to catchup in terms of latency, but the opportunity died on Arbitrum and picked up on other chains. 

### Takeaways

You might have read the above post and thought "what was the point of all that? why not just retweet the alpha from duoxehyon and walk away" and you're half right. The point of this post is not just to share that you can find crazy alpha to help you score profits, but rather that you should absolutely dive into interesting opportunities and see how far it can get you. You could see from the first iteration of my bot what kind of limited information I had on hand- I'm a solo indie anyway! I did what I could, tried to improve, found an alpha leak, used AI tools, and pieced together things in the most naive way possible. And somehow, it worked. It's not enough to read alpha leaks and move on, it still requires a lot of trial and error, but the value comes from putting your ego and fears aside and actually trying. In the end, you could have run a bot that does single block backrunning with no infrastructure other than a cheap Hetzner node for ~5 EUR a month! 

P.S. I'm open to chatting with other searchers! Whether you're a solo searcher looking to collab, or a team looking for an intern or short term contractor for some MEV-related work, send me a DM! I found it pretty tough to find any meaningful connections in MEV, and I'd like to break through and see what it's like to be part of a professional team or at least partner up. 
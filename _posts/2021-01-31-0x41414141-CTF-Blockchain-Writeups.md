---
title: ⛓ 0x41414141 CTF Blockchain Writeups 
author: Ashiq Amien
date: 2021-01-31 21:45:00 +0200
categories: [CTF, Blockchain]
tags: ctf
math: true
---

## Intro

This post is a writeup of all of the blockchain challenges hosted by the [0x41414141 CTF](https://ctfd.offshift.io). I was only planning on writing up one or two challenges, but I saw that many participants were struggling with getting round to using the blockchain and interact with the smart contracts so I figured it would be good to have a top-to-bottom writeup of all of the challenges.

## sanity-check

The first challenge was a sanity-check challenge, which is basically a way to make sure your basic tooling is setup correctly. This was the smart contract given:

```java
pragma solidity ^0.7.0;
//SPDX-License-Identifier: UNLICENSED

contract sanity_check {
    function welcome() public pure returns (string memory){
        return "flag{}";
    }
}
```

The only step is that you need to call the `welcome()` function to get the flag. The easiest way to do this is to load up this contract on [remix](https://remix.ethereum.org/) and use the `Load contract at address` function under the `Deploy & Run Transactions` tab. This contract was hosted on the Rinkeby network, so you would need to switch Metamask to this network before you load the contract. Once loaded, you should see something like the following:

![Desktop View](/assets/img/sample/0x41414141/sanity-check.png)

Clicking on the `welcome()` function returns the flag. Alternatively, you could write your own contract to interact with the deployed contract, as follows:

```java
pragma solidity ^0.7.0;
//SPDX-License-Identifier: UNLICENSED


contract get_sanity_back {
    
    function getflag() public pure returns (string memory) {
        address addy = 0x5CDd53b4dFe8AE92d73F40894C67c1a6da82032d;
        sanity_check sc = sanity_check(addy);
        return sc.welcome();
    }
}

contract sanity_check {
    function welcome() public pure returns (string memory) {
    }
}
```

Deploy the contract on Rinkeby and call the `getflag()` to get the flag. Since the function fetches a stored string, another alternative would be to [decompile the bytecode](https://rinkeby.etherscan.io/bytecode-decompiler?a=0x5cdd53b4dfe8ae92d73f40894c67c1a6da82032d) and just read the flag directly. 

## secure-enclave

We're given the following contract:

```java
pragma solidity ^0.6.0;

contract secure_enclave {
    event pushhh(string alert_text);

    struct Secret {
        address owner;
        string secret_text;
    }

    mapping(address => Secret) private secrets;

    function set_secret(string memory text) public {
        secrets[msg.sender] = Secret(msg.sender, text);
        emit pushhh(text);
    }

    function get_secret() public view returns (string memory) {
        return secrets[msg.sender].secret_text;
    }
}
```

At some point the `set_secret()` function was called with the flag as the secret, which stores the secret and emits an event with the secret. Even though the `secrets` mapping has a `private` visibility, it would still be possible to read the flag in this way as described [here](https://medium.com/coinmonks/how-to-read-private-variables-in-contract-storage-with-truffle-ethernaut-lvl-8-walkthrough-b2382741da9f). However, a much simpler option would just be to filter through the event logs and find the transaction with the flag in its logs. After filtering through the earlier transactions of the contract on [etherscan](https://rinkeby.etherscan.io/address/0x9b0780e30442df1a00c6de19237a43d4404c5237), we can find the [transaction](https://rinkeby.etherscan.io/tx/0x3e3498a9bbb97500f1cfb03fc4ce69aa2eddc475aaff8705414275065b8cb1ea#eventlog) with the log:
```
666c61677b33763372797468316e675f31735f4241434b4430305233445f303032307d0000000000000000000000000000000000000000000000000000000000
```

which decodes to `flag{3v3ryth1ng_1s_BACKD00R3D_0020}`.

## crackme.sol 

We're given the following contract:

```java
pragma solidity ^0.6.0;

contract crack_me{

    function gib_flag(uint arg1, string memory arg2, uint arg3) public view returns (uint[]) {
        //arg3 is a overflow
        require(arg3 > 0, "positive nums only baby");
        if ((arg1 ^ 0x70) == 20) {
            if(keccak256(bytes(decrypt(arg2))) == keccak256(bytes("offshift ftw"))) {
                uint256 check3 = arg3 + 1;
                if( check3< 1) {
                    return flag;
                }
            }
        }
        return "you lost babe";
    }

    function decrypt(string memory encrypted_text) private pure returns (string memory) {
        uint256 length = bytes(encrypted_text).length;
        for (uint i = 0; i < length; i++) {
            byte char = bytes(encrypted_text)[i];
            assembly {
                char := byte(0,char)
                if and(gt(char,0x60), lt(char,0x6E))
                { char:= add(0x7B, sub(char,0x61)) }
                if iszero(eq(char, 0x20))
                {mstore8(add(add(encrypted_text,0x20), mul(i,1)), sub(char,16))}
            }
        }
        return encrypted_text;
    }
}

```

To get the flag, we need to call `gib_flag()` with the correct parameters. The first argument needs to satisfy `(arg1 ^ 0x70) == 20`, which is the same as saying `arg1 = hex(20)^0x70`. The second argument was some string that decrypts to "offshift ftw". I copied over the `decrypt()` function on a separate contract and poked around to see what what the function was doing. I noticed that the decrypt function rotated the letters by 10 (i.e. ROT10), so it means we needed to apply ROT16 to "offshift ftw", which is "evvixyvj vjm". The last argument was needed some positive unsigned integer that when incremented returns a number less than 1. Since `uint` is an alias for `uint256`, it means that `2**256` will overflow. This means `arg3` should be `2**256-1` to pass through.

I wrote the following contract to solve the crack_me:

```java
pragma solidity ^0.6.0;

contract crack_me{

    function gib_flag(uint arg1, string memory arg2, uint arg3) public view returns (uint[] memory) {   
    }

    function decrypt(string memory encrypted_text) private pure returns (string memory){
    }
}

contract solve_me{
    
    function solve() public returns (uint[] memory) {
        address addy = 0xDb2F21c03Efb692b65feE7c4B5D7614531DC45BE;
        crack_me cm = crack_me(addy);
        
        uint256 arg1 = 100;
        string memory arg2 = "evvixyvj vjm";
        uint256 arg3 = uint(-1);
        uint[] memory ans = cm.gib_flag(arg1, arg2, arg3);
        return ans;
    }
    
}
	
```

Notice that my crack_me contract only has code stubs - that's because there were some compilation issues with the code given by the CTF host. For one, we'd need to drop the `flag` state variable and modify the return types to get everything to check out. I figured it would be easier to drop all the code, and fix the last error which was to change the return type of `gib_flag` from `uint[]` to `uint[] memory`. For some reason, the remix IDE kept freezing up with the returned uint array. However, I happened to have burp suite open from a previous challange, and I saw the response there:

```python
{"jsonrpc":"2.0","id":1011552430,"result":"0x00000000000000000000000000000000000000000000000000000000000000430000000000000000000000000000000000000000000000000000000000000030000000000000000000000000000000000000000000000000000000000000006e00000000000000000000000000000000000000000000000000000000000000670000000000000000000000000000000000000000000000000000000000000072000000000000000000000000000000000000000000000000000000000000004000000000000000000000000000000000000000000000000000000000000000370000000000000000000000000000000000000000000000000000000000000035000000000000000000000000000000000000000000000000000000000000005f000000000000000000000000000000000000000000000000000000000000005900000000000000000000000000000000000000000000000000000000000000300000000000000000000000000000000000000000000000000000000000000075000000000000000000000000000000000000000000000000000000000000005f0000000000000000000000000000000000000000000000000000000000000043000000000000000000000000000000000000000000000000000000000000005200000000000000000000000000000000000000000000000000000000000000400000000000000000000000000000000000000000000000000000000000000043000000000000000000000000000000000000000000000000000000000000004b00000000000000000000000000000000000000000000000000000000000000330000000000000000000000000000000000000000000000000000000000000044000000000000000000000000000000000000000000000000000000000000005f000000000000000000000000000000000000000000000000000000000000006d0000000000000000000000000000000000000000000000000000000000000033000000000000000000000000000000000000000000000000000000000000003800000000000000000000000000000000000000000000000000000000000000350000000000000000000000000000000000000000000000000000000000000034"}
```

Dropping the zero bytes leaves `43306e67724037355f5930755f435240434b33445f6d33383534`, which decodes to the flag: `C0ngr@75_Y0u_CR@CK3D_m3854`.

## Crypto Casino

We're given the following contract:

```java
pragma solidity ^0.6.0;

contract casino {

    bytes32 private seed;
    mapping(address => uint) public consecutiveWins;

    constructor () public {
        seed = keccak256("satoshi nakmoto");
    }

    function bet(uint guess) public {
        uint num = uint(keccak256(abi.encodePacked(seed, block.number))) ^ 0x539;
        if (guess == num) {
            consecutiveWins[msg.sender] = consecutiveWins[msg.sender] + 1;
        } else {
            consecutiveWins[msg.sender] = 0;
        }
    }

    function done() public view returns (uint16[] memory) {
        if (consecutiveWins[msg.sender] > 1) {
            return [];
        }
    }

}
```

To get the flag, we need at least 2 consecutive wins and then call `done()`. A transaction between several contracts need to all happen in the same block, so we can copy the `guess` and call `bet()` from another contract. I wrote the following code to do this:

```java
contract beatthehouse {
    
    bytes32 private seed;
    
    function allonred() public {
        seed = keccak256("satoshi nakmoto");
        address addy = 0x186d5d064545f6211dD1B5286aB2Bc755dfF2F59;
        casino grandwest = casino(addy);
        uint guess = uint(keccak256(abi.encodePacked(seed, block.number))) ^ 0x539;
        grandwest.bet(guess);
    }
    
    function getWins() public view returns (uint) { 
        address addy = 0x186d5d064545f6211dD1B5286aB2Bc755dfF2F59;
        casino grandwest = casino(addy);
        return grandwest.consecutiveWins(address(this));
        
    }
    
    
    function submitwins() public view returns (uint16[] memory) {
        address addy = 0x186d5d064545f6211dD1B5286aB2Bc755dfF2F59;
        casino grandwest = casino(addy);
        uint16[] memory flag = grandwest.done();
        return flag;
    }
}
```

The function `allonred()` calculates the same number as the casino contract, ensuring that we'll win each time. My tx's all went through, but for some reason my `consecutiveWins` weren't being counted. I tried triple checked my code and redeployed a few times but got the same result.

Since I wasn't interacting with a verified contract on etherscan, I decided to have a look at the contract [bytecode](https://rinkeby.etherscan.io/bytecode-decompiler?a=0x186d5d064545f6211dD1B5286aB2Bc755dfF2F59). After looking around, I saw that bet() wasn't using `msg.sender`, but rather `tx.origin`, which is the EOA (externally owned account) associated to any smart contract call:

```ruby
def bet(uint256 _option) payable: 
  require calldata.size - 4 >= 32
  if 1337 xor sha3(stor0, block.number) != _option:
      unknown143af907[tx.origin] = 0
  else:
      unknown143af907[tx.origin]++
```

Loading up the original contract on remix, I checked the consecutive wins with my EOA address directly and found that I already had 7 wins. After calling the `done()`, I ran into an issue similar to the crack_me return. Again using burp to proxy the response, we can get the output `666c61677b4433434e3752406c315a33445f434035314e30535f3575636b3533317d`, which decodes to `flag{D3CN7R@l1Z3D_C@51N0S_5uck531}`. This is a great example of why you shouldn't just ape into unverified contracts :D 

## Rich Club

We're given the following contract:

```java
pragma solidity ^0.6;
//SPDX-License-Identifier: MIT

interface ERC20 {
    function balanceOf(address account) external view returns (uint256);
}

contract RICH_CLUB {

    ERC20 UNI;
    event new_member(string pub_key);
    event send_flag(string pub_key, string flag);

    constructor() public {
        UNI = ERC20(0x1f9840a85d5aF5bf1D1762F925BDADdC4201F984);
    }

    function grant_membership(string memory _pub_key) public {
        require(bytes(_pub_key).length > 120, "invalid public key");
        require(UNI.balanceOf(msg.sender) >= 6e20, "you don't look rich to me");
        emit new_member(_pub_key);
    }

    function grant_flag(string memory _pub_key, string memory encoded_flag) public {
        require(msg.sender == address(0x30cE246A1282169895bf247abaE77BA69d5B2416), "you don't have access to this");
        emit send_flag(_pub_key, encoded_flag);
    }
}
```

In this challenge we need to provide a public key and a bot will return an encrypted flag for us to decrypt with the corresponding private key. In this way, we can't snoop around in the contract storage or event logs to find the flag. First, I used [this site](https:// asecuritysite.com/encryption/ethadd) to generate a public/private key pair for me. I then needed to get 600 UNI tokens before I could call the `grant_membership()` function. (Notice that UNI has 18 decimal places, so `6e20` is `600e18`, i.e. 600 tokens). I got some rinkeby ETH from the [authenticated faucet](https://faucet.rinkeby.io/) and bought some UNI on the Rinkeby Uniswap. I noticed that other users were using FlashSwaps to get their 600 UNI for the transaction, but I already had my UNI so I carried on. (If you'd like to see a flashswap demonstration, let me know in the comments!) After calling `grant_membership()` the admin/bot sent the transaction with the encrypted flag in the [event logs](https://rinkeby.etherscan.io/tx/0xf0cfa8cd8da7eaa5a9c52bb7407d55d0d69792eb23dd46feb371bf684ef48cb1#eventlog).

At this point I wasn't really sure which tool to use to decrypt the flag. After a few hours of digging and poking around I came across [https://github.com/ecies/py](https://github.com/ecies/py) which could be used to decrypt my flag. Loading the private key and running the following in eciespy gives us the flag:

```python
>>> encflag=b"\x04\xd1\x1f\xbd\x8c\xebe\xf4\xec\\\xce#\xb2\xa6>\xaf\x05|\xec\xad?X\xb3\xa2\xbe\xf8\xc5\x1dg*r\xe7\x00d\x9d\x18\xb7\xda\xb6\xd2'\xa6\xdc\x84[\x1a\xc07j\xba5\n|x\x18\xd6\xe7\xb7\x90\xf1,\x02\xb7&d\xd7\xd0\x99\xe1\x00,\x8f\n\xb7\x13\x17\xf8\xdd\x1f\xa0\x8a\x7f=]{\xf3\xacm\x8as_\x9d\xee\xdd0L\xb5\n)\xe4A/\xbd\x82\x14\xf1a\xdc\x80\xa4\xfb9M\xd7>\x94o\xf3\xeeo\x0e,\xcb\x12\xd7\xcf\x17*K\x15\xad"
>>> decrypt(sk_hex, encflag)
b'flag{l0@ns_ar3nt_7ha7_b@d_tbh8877}'
```

A big thanks to the team behind this CTF! I really enjoyed these, and it's cool to see that more CTFs are incoporating blockchain-style challenges :)
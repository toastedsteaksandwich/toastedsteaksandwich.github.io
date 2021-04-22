---
title: ALLES! CTF 2020 - Prehistoric Mario Writeup
author: Ashiq Amien
date: 2020-09-07 21:45:00 +0200
categories: [CTF, Reverse Engineering]
tags: ctf
---

Prehistoric mario was a reverse engineering challenge from the [ALLES! CTF](https://play.allesctf.net/). We are given an APK and the hint is that we need to trigger the right boxes to get the flag. The game is a platformer with 11 question mark boxes, and 4 colours per box:

![Prehistoric Mario](/assets/img/sample/ALLES-front.jpg)



I started by decompiling the app with `d2j-dex2jar` and found a `checkFlag()` method. 

![checkFlag() decompiled](/assets/img/sample/ALLES-code.png)

This method initializes a byte array of size 11 and populates it with the state of each of the question mark blocks. We then get the SHA-256 digest of the byte array and salt `P4ssw0rdS4lt`. If the hash matches the hardcoded hash, an encrypted text file is decrypted with the byte array as the key. The decrypted file is used to update the map with the flag. 

## Objective

To get the flag, we need to crack the hardcoded hash. This means finding the message format, and then brute-forcing all `11^4` combinations of block states.

After examining the smali code of the app, I found that the message just consisted of the byte array and the salt, and not the tiledMapTileLayer as indicated by the decompiled code. To make sure I was on track, I added the following smali logging code to see what hash we are getting without hitting any blocks:

```
.line 254

    invoke-static {v8}, Lcom/alles/platformer/MyPlatformer;->toHex([B)Ljava/lang/String;

    move-result-object v8    

    const-string v9, "test hash 024800ace2ec394e6af68baa46e81dfbea93f0f6730610560c66ee9748d91420"
    
    invoke-static {v8, v9}, Landroid/util/Log;->d(Ljava/lang/String;Ljava/lang/String;)I
```

After rebuilding, resigning and reinstalling the app, I received the following log:

```
alles-ctf-2020$ adb logcat | grep 024800ac
...
96832b93cf0184fe129e9df46ed07e84ca8e2426d11391b9959d364026807dec: test hash 024800ace2ec394e6af68baa46e81dfbea93f0f6730610560c66ee9748d91420
``` 

## Finding the message

We get the `96832b93` hash without hitting any blocks, so presumably the byte array is populated with zeroes and not any of the values as per the `assets/map.tmx` file in the APK. I recreated the message using a byte-array of 11 zeroes, salted it and digested it using a separate java file: 

```
...
String data = "P4ssw0rdS4lt";
MessageDigest messageDigest;
messageDigest = MessageDigest.getInstance("SHA-256");
messageDigest.update(arrayOfByte);
messageDigest.update(data.getBytes());
...
```

The hash produced from the above code was different to the expected `96832b93` hash. Since the code was pretty much identical, and the salt was the same, it meant that the blocks were updating somehow. I added more smali logging code to log the values when the array was updated, and the following was logged:

```
... D ALLES-CTF this is the value: 57
... D ALLES-CTF this is the index: 0
```

If we look at the `assets/map.tmx` file in the APK, we see that the `questionmarkType` blocks have values 0, 21, 37, 97 and 1337. The 1337 block corresponded to the rainbow block used for checking the flag, and the others represented the state of the block. I'm not sure where the 57 value came from since it wasn't the same number as any state. I checked this against my local `MessageDigest` java file by updating the byte at index 0 to 57, and received the same `96832b93` hash, so we found the correct message format. 

## Cracking the hash

At this point I was pretty uncertain about the brute-force because of the `57` value, but went ahead with it. I brute-forced all `11^4` combinations of the byte array and digested it, and this was the outcome: 

```
alles-ctf-2020$ java MessageDigestExample | grep 024800ace2ec394e6af68baa46e81dfbea93f0f6730610560c66ee9748d91420 -B 1
...
data:[21, 0, 97, 37, 21, 37, 37, 97, 97, 37, 21]
digested:024800ace2ec394e6af68baa46e81dfbea93f0f6730610560c66ee9748d91420
```

The hash is cracked!

## Loading the combination

We found the block combination to find the flag. I figured the order of the blocks was read in order from left to right, and that the the first hit corresponded to 21, second to 37 and third to 97. but this did not seem to work for some reason. I figured it would be time consuming to check all of those things, so I wanted to manually populate the array in the smali code. Doing it byte-for-byte was finicky since I had to deal with several registers in the process, and then do that 11 times (once for each byte). I saw that the array is initialized with zeroes in the smali code, so I edited this array to correspond with the data values as follows:

```
.array-data 1
  0x15t 
  0x0t
  0x61t
  0x25t
  0x15t
  0x25t
  0x25t
  0x61t
  0x61t
  0x25t
  0x15t
.end array-data
```

And just incase that weird `57` value popped up again, I reloaded these values just before the digest:

```
fill-array-data v3, :array_0
   
invoke-virtual {v6, v3}, Ljava/security/MessageDigest;->update([B)V
```

Hitting the rainbow block after rebuilding, resigning and reinstalling the app just one more time gives us the flag :)

![flag!](/assets/img/sample/ALLES-flag.jpg)

I'm not sure if I missed the intended path to the solution, but I learnt a ton and had a lot of fun. Big thanks to the team behind ALLESCTF and 0x4d5a for creating the challenge!

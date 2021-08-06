---
title: 🚩 DownUnder CTF 2020 - Flag Getter Challenge Writeup
author: Ashiq Amien
date: 2020-09-20 16:00:00 +0200
categories: [CTF, Reverse Engineering]
tags: ctf
---

We're given an Android app with the hint "An app that gets the flag for you! What more could you possibly want?". The app has 4 main buttons, and after each button press we get a "Response Code: 200" message. This indicates that the flag is being fetched from a server. Setting up a proxy and trying to intercept the request does not work:

![Cert Pin](/assets/img/sample/DownUnder-Proxy.png)

This means there must be some kind of certificate pinning happening which sees that the certificate of our proxy server does not match the hard-coded certificate or certificate pin. To look under the hood, I decompiled the app with `d2j-dex2jar` and had a look at the jar file with jd-gui. 

![Under the Hood](/assets/img/sample/DownUnder-Obfuscated.png)


## Objective

We see that just about everything is minified, which makes it harder to understand what's going on in the code.  I initially tried a few of the standard cert pin workarounds like using universal SSL pin bypass scripts with Frida. None of those worked, but I realized that the app name and challenge name was a hint that we are loading a GET request to get the flag. If we get the full URL with the GET parameters we should have the flag. 


I tried patching the application to log several areas of the app to try and get the full URL. I used the following log method:

```
    invoke-static {v0}, Landroid/util/Log;->d(Ljava/lang/String)I
```

However, this did not work for some reason. I guess some of the Log methods were obfuscated as well. As I was trying to patch the app, I saw the following code:

```
    const-string v2, "http:"`

    invoke-static {v2}, Lb/a/a/a/a;->g(Ljava/lang/String;)Ljava/lang/StringBuilder;

    move-result-object v2
```

This StringBuilder object was used to build the URL. Looking around a bit more, I saw that class `e.b0` had a toString() method that looked like it was using a StringBuilder object to build URL. 

![e.b0 toString](/assets/img/sample/DownUnder-toString.png)

## Using the StringBuilder object

I wrote a script to hook the `e.b0.toString()` method to print the return value, but the hook was not caught anywhere for some reason. Even though this method hook was not being caught, we could use the StringBuilder object and get the toString() directly. Using the following Frida script, we can hook the toString() method and print out the URL directly:

```
Java.perform(function toastedsteaksandwich() {                

    var toStringHook = Java.use('java.lang.StringBuilder');

    toStringHook.toString.implementation = function(){
    	var url = this.toString();
    	console.log("here is the url: " + url);
    	return url;
    }

},0);
```

Save the script as `logurl.js` and run `frida -U -f com.example.get_flag -l logurl.js --no-pause`. After pressing one of the four buttons, we can see the URL being built:

![StringBuilder URL](/assets/img/sample/DownUnder-Log.png)

Navigating to the URL gives us a piece of the flag. We can get the full flag by navigating to each of the 4 URLs and concatenating the result to get the flag: `DUCTF{n0t_s0_s3cre7_4ft3r_4LL_!!11!}`. 

I didn't have too much time to play this CTF so I submitted this flag a few hours after the deadline, but I figured it was worth a try and a writeup anyway. Big thanks to DownUnderCTF and the author for the fun challenge :) 







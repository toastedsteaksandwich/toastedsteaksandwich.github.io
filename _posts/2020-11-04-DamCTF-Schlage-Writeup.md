---
title: DamCTF 2020 - Schlage Challenge Writeup
author: Ashiq Amien
date: 2020-11-04 21:00:00 +0200
categories: [CTF, Reverse Engineering]
tags: ctf
---

We're given a binary with a description "I went to the hardware store yesterday and bought a new lock, for some reason it came on a flash drive. Can you figure out how to unlock it? I really need to get into my apartment." When running the binary, we see a lock with 5 pins. We need to solve each pin to unlock the lock and get the flag. The pins are unlocked in order 3 - 1 - 5 - 2 - 4, which can be determined by trial-and-error. 

## Pin 3

Dropping the binary into hopper (a disassembler), we can see the following pseudocode generated for pin 3:


```
int do_pin3() {
    if ((*(int32_t *)dword_202018 != 0x0) && ((*(int8_t *)(sign_extend_64(*(int32_t *)dword_202018 - 0x1) + pins) & 0xff ^ 0x1) != 0x0)) {
            rax = puts("Hmm, this pin won't budge!");
    }
    else {
            puts("Give me a number!");
            if ((0xffffffffdeadbeef ^ get_int()) == 0x13371337) {
                    *(int8_t *)byte_20203b = 0x1;
                    rax = puts("Great!");
            }
            else {
                    rax = puts("Nope!");
            }
    }
    return rax;
}
```

We can see that our input XORed with `0xffffffffdeadbeef` needs to equal `0x13371337`. Since anything XORed with itself is 0, we can work out that our input needs to be `0xffffffffdeadbeef` XOR `0x13371337`, which is just `3449466328` as an integer.

## Pin 1

The pseudocode for pin 1 is shown below:

```
int do_pin1() {
    if ((*(int32_t *)dependencies != 0x0) && ((*(int8_t *)(sign_extend_64(*(int32_t *)dependencies - 0x1) + pins) & 0xff ^ 0x1) != 0x0)) {
            puts("Hmm, this pin won't budge!");
    }
    else {
            puts("Number please!");
            var_16 = 0x0 ^ get_int() & 0xff;
            for (var_14 = 0x0; var_14 <= 0x5; var_14 = var_14 + 0x1) {
                    var_16 = var_16 ^ *(int8_t *)(rbp + (sign_extend_32(var_14) - 0xe)) & 0xff;
            }
            if (var_16 == 0xee) {
                    *(int8_t *)pins = 0x1;
                    puts("Great!");
            }
            else {
                    puts("Nope!");
            }
    }
    rax = *0x28 ^ *0x28;
    if (rax != 0x0) {
            rax = __stack_chk_fail();
    }
    return rax;
}
```

I got stuck here for a while, mainly because I'm not very experienced with reverse engineering linux binaries. I could see that our answer needed to be some number between 0 and 256, and I used gdb for some tracing, but overall I couldn't directly answer this problem. I figured I could try another route. I came across [@leonjza](https://twitter.com/leonjza)'s [Frida-boot workshop](https://www.youtube.com/watch?v=CLpW1tZCblo) to instrument linux binaries. Since I was already familiar with using Frida to instrument mobile applications, I figured it would not be too much of a stretch to learn how to use Frida for linux binary instrumentation.

The workshop was highly informative and I'd highly recommend giving it a watch. I learnt about instrumenting the binary by hooking it's functions, similarly to how you'd do it for mobile apps. However, I couldn't directly apply the knowledge to the above problem since `do_pin1()` did not take arguments. Moreover, we needed to solve the actual challenge and not just modify a return value to get through to the next level. The workshop gave me a kick to do some extra research, and I found that you could use Frida to directly modify in-memory code to suit your needs. This meant I could patch in my own loop to brute-force the code, even if the function did not have arguments. 

### Pin 1 - Restarting with Frida

To brute-force the answer, we to submit numbers between 0 and 256 and check if we get the answer. Consider the following steps:

- We need to loop inside the `do_pin1()` function itself since it would be a hassle to call `do_pin1()` each time. To make a loop inside the code, I looked for the failed condition of the pin and patched it to loop back to the start of the pin. 

![Loop patch](/assets/img/sample/schlage-loop-patch.jpg)

- We don't really want to be typing in 256 numbers, even if the numbers are corresponding to values in the brute-force. To work around this, we can overwrite the call to `get_int` and manually inject our current value for brute-force.

- Once we have the right answer, we can remove our fake loop and let things run as usual. (In hindsight, a well placed loop would not need this step, but it worked regardless)

The resulting Frida script is as follows:

```javascript
var pin1 = Process.getModuleByName('schlage')
console.log("base " + pin1.base.toString());

//The variable we're going to use for the brute-force
var i = 0;

//Patched loop into code
var failedcall = pin1.base.add(0xe7b);
var newentry = pin1.base.add(0xe03);
Memory.patchCode(failedcall, 2, function (code) {
	var cw = new X86Writer(code, { failedcall: failedcall });
	cw.putJmpAddress(newentry);
	cw.flush();
	console.log("patched in a loop between " + failedcall + " and " + newentry);
});

//Hook just before our call to get_int
var beforepin1getint = pin1.base.add(0xe0f);
Interceptor.attach(beforepin1getint, {
	onEnter: function (args) {
        
        	//Move our own result into the get_int destination register
		if(i<256){
			console.log("Hook caught, patching the pin in..."); 
          		var pc = pin1.base.add(0xe14);
			Memory.patchCode(pc, 5, function (code) {
			  var cw = new X86Writer(code, { pc: pc });
			  const j = i;
			  cw.putMovRegU32('eax', j);
			  cw.flush();
			  console.log('Patched code with: ' + i.toString());
			  i++;
			});
		}

	}	

});

//Once we have a pin, restore the usual flow of the code
var foundpin = pin1.base.add(0xe4d);
Interceptor.attach(foundpin, {
	onEnter: function (args) {
		console.log("Found pin! the pin is " + (i-1).toString());
		Memory.patchCode(failedcall, 2, function (code) {
			var cw = new X86Writer(code, { failedcall: failedcall });
			cw.putJmpAddress(pin1.base.add(0xe82));
			cw.flush();
			console.log("Restored the loop, executing as usual... ");
		});
	}		

});
	

```

This gives us the following output from Frida:

```
[+] hook caught, patching the pin in...
[+] patched code with: 94
[+] hook caught, patching the pin in...
[+] patched code with: 95
[+] hook caught, patching the pin in...
[+] patched code with: 96
[+] hook caught, patching the pin in...
[+] patched code with: 97
[+] hook caught, patching the pin in...
[+] patched code with: 98
[+] hook caught, patching the pin in...
[+] patched code with: 99
[+] found pin! the pin is 99
[+] restored the loop, executing as usual... 
```

Pin 1 is solved! 

```
Number please!
Nope!
Number please!
Nope!
Number please!
Nope!
Number please!
Great!

          ------------
         /            \
        /   --------   \
       /   /        \   \
      /   /          \   \
    ========================
    |                      |
    |    |===>   <====|    |  Pin 1
    |    |=====><=====|    |  Pin 2
    |    |===>   <====|    |  Pin 3
    |    |=====><=====|    |  Pin 4
    |    |=====><=====|    |  Pin 5
    |                      |
    ========================

```


## Pin 5

Next up was pin 5, which was a lot easier to solve than the previous pin. The pseudocode is shown below:

```
int do_pin5() {
    if ((*(int32_t *)dword_202020 != 0x0) && ((*(int8_t *)(sign_extend_64(*(int32_t *)dword_202020 - 0x1) + pins) & 0xff ^ 0x1) != 0x0)) {
            rax = puts("Hmm, this pin won't budge!");
    }
    else {
            srand(0x42424242);
            puts("I bet you can't guess my random number!");
            if (get_int() == rand()) {
                    rax = puts("Wow! That was some impressive guessing.");
                    *(int8_t *)byte_20203d = 0x1;
            }
            else {
                    rax = puts("Good try!");
            }
    }
    return rax;
}

```

Here we have a hardcoded seed, so we just need to see what the outcome of rand is. Since we're using Frida, we can hook `rand()` and then replace our input with the outcome. Since the value is always going to be the same, I nopped out the call to `get_int` and replaced it with hardcoded value which I got after running the program with the `rand()` hook. When solving the actual challange binary, all we need is the number generated according to the seed.

```javascript
var pin5 = Process.getModuleByName('schlage');
console.log("base " + pin5.base.toString());

var randPtr = DebugSymbol.getFunctionByName("rand");
var rand = new NativeFunction(randPtr, "int",[]);

//Intercept rand() after the number is generated and print it
Interceptor.attach(rand,{
	onLeave: function(retval){
		console.log("generated random number: " + retval.toInt32());
	}
});

//Patch the found value as our input
var patchrand = new NativePointer(pin5.base.add(0xed8));
Memory.patchCode(patchrand, 5, function (code) {
	var cw = new X86Writer(code, { patchrand: patchrand });
	cw.putMovRegU32('eax', 1413036362);	
	cw.flush();
	console.log("patched your input as 1413036362 for next time");
});

//Nop out the call that fetches new input
var nopinput = new NativePointer(pin5.base.add(0xedd));
Memory.patchCode(nopinput, 5, function (code) {
	var cw = new X86Writer(code, { nopinput: nopinput });
	cw.putNopPadding(5);	
	cw.flush();
	console.log("nopped out fetching input");
});

```

## Pin 2

Pin 2 was similar to pin 5 in that we needed to call `rand()` with a specific seed. The difference was that the seed was the return from time(), which meant the answer was different each time we ran the binary. The pseudocode for pin 2 is below:

```
int do_pin2() {
    if ((*(int32_t *)dword_202014 != 0x0) && ((*(int8_t *)(sign_extend_64(*(int32_t *)dword_202014 - 0x1) + pins) & 0xff ^ 0x1) != 0x0)) {
            rax = puts("Hmm, this pin won't budge!");
    }
    else {
            var_8 = time(0x0);
            puts("Hmm, a little piece of paper just fell out of the lock with some random numbers on it!");
            puts("I wonder what it means?");
            printf(0x15d7);
            srand(var_8);
            puts("What's your favorite number?");
            if (get_int() == rand()) {
                    rax = puts("Woah, that's the lock's favorite number too! Small world, eh?");
                    *(int8_t *)byte_20203a = 0x1;
            }
            else {
                    rax = puts("To each their own I guess!");
            }
    }
    return rax;
}

```

To solve this pin, I placed a code hook after `srand` was set with the time. Once attached, I called `rand()` directly and reseeded `srand` with the original time seed, so that when we go back to solve the pin, the random number will be the same as the one that we caught.



```javascript
var pin2 = Process.getModuleByName('schlage');
console.log("base " + pin2.base.toString());

var time = 0;
var randPtr = DebugSymbol.getFunctionByName("rand");
var rand = new NativeFunction(randPtr, "int",[]);

var srandPtr = DebugSymbol.getFunctionByName("srand");
var srand = new NativeFunction(srandPtr, 'void', ["int"]);

var beforeourinput = pin2.base.add(0xfa4)
Interceptor.attach(beforeourinput,{
	
	onEnter: function(args) {

        	//call rand() and print the answer
		console.log("this is your answer: " + rand());

        	//set the seed back so that when we call rand() again it's the same answer as above
		srand(time);
		console.log("reseeded with: " + time);
	}

});

//Intercept and save the seed
var timecall = DebugSymbol.getFunctionByName("time");
Interceptor.attach(timecall,{
	
	onLeave: function(retval) {
		console.log("this is the time used as the seed: "  + retval.toInt32());
		time = retval.toInt32();
	}	

});
```


When calling pin 2 with the above script, Frida prints the following:
```
Attaching...                                                            
base 0x55f76a5db000
[Local::schlage]-> this is the time return: 1604438721
this is your answer: 1133922972
reseeded with: 1604438721
```

Submitting the answer `1133922972` solves this pin:
```
Which pin would you like to open?
> 2
Hmm, a little piece of paper just fell out of the lock with some random numbers on it!
I wonder what it means?
1604438721
What's your favorite number?
> 1133922972
Woah, that's the lock's favorite number too! Small world, eh?
```
When solving the actual challenge binary to get the flag, you'd need to run the binary locally and manually modify the seed in the script to be the time as given by the challenge binary. 

## Pin 4

The pseudocode for pin 4 is below:

```
int do_pin4() {
    if ((*(int32_t *)dword_20201c != 0x0) && ((*(int8_t *)(sign_extend_64(*(int32_t *)dword_20201c - 0x1) + pins) & 0xff ^ 0x1) != 0x0)) {
            puts("Hmm, this pin won't budge!");
    }
    else {
            var_30 = 0x0;
            puts("What's your favorite sentence?");
            fgets(&var_30, 0x20, *stdin@@GLIBC_2.2.5);
            rax = strcspn(&var_30, 0x1677);
            *(int8_t *)(rbp + (rax - 0x30)) = 0x0;
            rcx = rand();
            var_38 = (rcx - ((SAR(HIDWORD(rcx * 0x66666667), 0x2)) - (SAR(rcx, 0x1f)) << 0x2) + ((SAR(HIDWORD(rcx * 0x66666667), 0x2)) - (SAR(rcx, 0x1f))) + ((SAR(HIDWORD(rcx * 0x66666667), 0x2)) - (SAR(rcx, 0x1f)) << 0x2) + ((SAR(HIDWORD(rcx * 0x66666667), 0x2)) - (SAR(rcx, 0x1f)))) + 0x41;
            var_40 = 0x0;
            rax = strlen(&var_30);
            var_34 = rax;
            for (var_3C = 0x0; var_3C < var_34; var_3C = var_3C + 0x1) {
                    var_40 = var_40 + (sign_extend_64(*(int8_t *)(rbp + (sign_extend_32(var_3C) - 0x30)) & 0xff) ^ var_38);
            }
            if (var_40 == 0x123) {
                    puts("Such a cool sentence!");
                    *(int8_t *)byte_20203c = 0x1;
            }
            else {
                    puts("Not a big fan of that sentence");
            }
    }
    rax = *0x28 ^ *0x28;
    if (rax != 0x0) {
            rax = __stack_chk_fail();
    }
    return rax;
}
```

There's definitely some weird looking things in here. After spending some time poking around, I found that `var_38` depended on the random numbers generated from pin 2 and was always around the range of `0x40` to `0x4a`. I knew that we need to submit some set of characters (a sentence), and that we looped over it for some result to be 0x123. Other than that, I wasn't too sure what was going on. I patched in another loop and tried random sentences, similar to what I did with pin 1, but that did not work. 

After spending lots of time going nowhere, I took the hint that `(sign_extend_64(*(int8_t *)(rbp + (sign_extend_32(var_3C) - 0x30)) & 0xff)` fetches a character of our string. From here, it was clear that we are looping over our string and taking the sum of each character XOR  `var_38`. The challenge was made clear: find some set of characters `c1, ..., cn` such that  `c1 XOR var_38 + ... + cn XOR var_38 = 0x123`. The answer could always be found through brute-force, as shown in the frida script below:

```javascript
var pin4 = Process.getModuleByName('schlage');
console.log("base " + pin4.base.toString());

var seed = 0;
var solved = false;
var found = "nothing found";
var randPtr = DebugSymbol.getFunctionByName("rand");
var rand = new NativeFunction(randPtr, "int",[]);
var srandPtr = DebugSymbol.getFunctionByName("srand");
var srand = new NativeFunction(srandPtr, 'void', ["int"]);

/*
We attach this script before we solve pin 2 so we can intercept the time seed. 
The snippet below patches pin 2 so that we jump to the success condition regardless of the input. 
This is not needed if we have one script with all pin solves or if we manually add the seed to the script if we have it.
*/
var pin2 = Process.getModuleByName('schlage');
var pin2bypass = pin2.base.add(0xfb9);
Memory.patchCode(pin2bypass, 2, function(code) {
	pin2bypass.writeByteArray([0x74,0x15]);
});

//Intercept and save the seed from pin 2
var timecall = DebugSymbol.getFunctionByName("time");
Interceptor.attach(timecall,{
	
	onLeave: function(retval) {
		console.log("this is the time used as the seed: "  + retval.toInt32());
		seed = retval.toInt32();
	}	

});

//Attach to the binary once we have var_38
var getvar38 = pin4.base.add(0x10b4);
Interceptor.attach(getvar38, {

	onEnter: function(args) {		
		var targetans = parseInt('123',16);
		console.log("this is our target: " + targetans);
		var answers = parseInt("aaaa", 36);
		
		while(!solved){
           		//Calculate c1 XOR var_38 + ... + cn XOR var_38 
           		var tot = 0;
			for(var k = 0; k < answers.toString(36).length; k++){
				tot = tot + (answers.toString(36).charCodeAt(k) ^ parseInt(this.context.rax,16));
			}
			
           		 //Test if c1 XOR var_38 + ... + cn XOR var_38 = 0x123
			if(tot == targetans){			
				solved = true;
				found = answers.toString(36);
				console.log("found answer: " + found);			
				
              		         //Reseed and recall rand so we when we submit the answer next time the same random number will be generated
				srand(seed);
				rand();	
				console.log("reseeded and recalled rand()");					
			}
          	        //If we didn't find an answer, move to the next sequence of characters
			answers = answers + 1;
		}
	}
}); 

```


After submitting a random string to run the script, this was the output from frida with the above script attached:
```
[Local::schlage]-> this is the time return: 1604439979
this is our target: 291
found answer: at6v
reseeded rand()

```
Calling pin 4 again and submitting the found answer `at6v` solves the pin, and the challenge:

```
Which pin would you like to open?
> 4
What's your favorite sentence?
at6v
Such a cool sentence!

                        ------------
                       /            \
                      /   --------   \
                     /   /        \   \
                    /   /          \   \
    ========================
    |                      |
    |    |===>   <====|    |  Pin 1
    |    |===>   <====|    |  Pin 2
    |    |===>   <====|    |  Pin 3
    |    |===>   <====|    |  Pin 4
    |    |===>   <====|    |  Pin 5
    |                      |
    ========================

Congratulations on opening the lock! I really needed to get back into my apartment.
```

This challenge definitely took much longer than I expected since I detoured into learning more about Frida, but I'm glad I did since it's quite a fun tool to use. Big thanks to [@leonjza](https://twitter.com/leonjza) for the great [frida workshop](https://www.youtube.com/watch?v=CLpW1tZCblo)! I would highly recommend checking it out if you interested to get your hands dirty with Frida.

Overall I found this challenge to be a bit tough, mostly due to pins 1 and 4. I thought this challenge would be straight forward since it was a beginner challenge, but it probably would have been better to learn some C and or practice easier challenges before taking on this one. I've left open the comments in case there is anything anyone wants to raise or have suggestions on how I could have done things differently. Regardless, big thanks to the author of this challenge! I've learnt a lot. :) 


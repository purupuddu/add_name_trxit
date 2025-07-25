---
title: "CodeFest CTF 2017 - Ricks' Secure Scheme Writeup"
date: '2017-09-23'
lastmod: '2023-07-03T19:19:24+02:00'
categories:
- writeup
- codefest17
tags:
- pwn
- rand
authors:
- anticlockwise
---

## Capture United States - Ricks’ Secure Scheme


<img class="img-responsive" src="/codefest17/USA-1.png" alt="Screenshot of service menu showing login option and usage logs" width="603" height="286">


The schwifty_service that runs under the port 10987 of the addressed server presents us a menu interface with some options given:



```
1. Login with the password (*)
2. View the secret contents (#).
3. View the usage logs.
4. Exit.
* => It’s theoretically impossible that you can login
# => Requires login.
```

To view the secret content we must login first.

At first let's search the login procedure in IDA Pro. Decompiling the code in the only function called by main, we can find:

<img class="img-responsive" src="/codefest17/USA-2.png" alt="Screenshot of decompiled function calling sub_400A26()" width="603" height="244">

So let's go have a look at `sub_400A26`

<img class="img-responsive" src="/codefest17/USA-3.png" alt="Screenshot of sub_400A26() decryption function using time() and rand()" width="603" height="389">

This function looks like a decryption function! We can notice that it initializes the random seed with `time()` and then xores the first 34 chars of the password with the random value obtained each time with `rand()`.

The decrypted password is then passed to the `strncmp` function, against the bytes at address 400E60

At that address there are `bytes = [0x33, 0x12, 0x46, 0x67, 0xF6, 0x2B, 0x5A, 0x2E, 0x5B, 0x7E, 0xD6, 0xF7, 0xA2, 0x33, 0xD5, 0x7A, 0x87, 0x39, 0x5F, 0x92, 0x73, 0xF5, 0xB1, 0xA5, 0x81, 0xB0, 0x6A, 0x84, 0x38, 0xCD, 0x9B, 0xEA, 0x99, 0xDA]` followed by the string `'Welcome to my...\x00'`

So how can we pass the check? If we could obtain the same seed and then xor the `bytes` with the same pseudo-random sequence, then we will have a string, that, when re-xored in the function, will produce the original bytes!
Fortunately the `time()` function has a granularity of 1 second, so we have this whole second to compute the string and pass it to the server.

Let's make a script:
```python
from pwn import *
import ctypes
import hashlib

libc = ctypes.CDLL("libc.so.6")
passwd = [0x33, 0x12, 0x46, 0x67, 0xF6, 0x2B, 0x5A, 0x2E, 0x5B, 0x7E,
0xD6, 0xF7, 0xA2, 0x33, 0xD5, 0x7A, 0x87, 0x39, 0x5F, 0x92, 0x73, 0xF5,
 0xB1, 0xA5, 0x81, 0xB0, 0x6A, 0x84, 0x38, 0xCD, 0x9B, 0xEA, 0x99, 0xDA]

def genPassword(time):
    libc.srand(time)
    s = ""
    for i in xrange(34):
        s += chr(passwd[i] ^ (libc.rand() % 255))
    return s


s = genPassword(1505768803) + 'Welcome to my...\x00'
print s
quit()


#p = process("./schwifty_service")
p = remote("13.126.83.119",10987)

while True:
    print p.readuntil("Requires login.");

    p.sendline("1")

    print p.readuntil("Enter the password: ");

    s = genPassword(libc.time(0))
    s+= 'Welcome to my...\x00' #REMEMBER TO CONCATENATE THE REST OF THE STRING

    p.sendline(s)
    print "[] password sended", s

    l = p.readline()
    print l

    if l == "I told you you couldn't login!\n":
        continue
    break

p.interactive()

```

With `import ctypes` we are sure that we are using the same implementation of `rand()` as the libc to get the same sequence.

Let's run it, and... It works!
```
What?!?!?! How?!!!! Anyway, you've successfully logged in!
```

Printing the secret will give us an epoch integer, and the md5 of that integer will give us the flag!

```
flag{c51edda648c6949638488457f32874d6}
```

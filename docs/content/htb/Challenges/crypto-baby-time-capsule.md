---
authors:
    - Lukas Daubner
date: 16-01-2024
---

# Crypto - Baby Time Capsule

You are provided with a code that generates encoded message + public key, using RSA-1024.

The problem is that each time, the same exponent (e) is used and it is also a low one `e=3`. That means that it can be cracked using Chinese Reminder Theorem See for more details: https://crypto.stackexchange.com/questions/6713/low-public-exponent-attack-for-rsa https://asecuritysite.com/rsa/rsa_ctf02

The point is ask the server few times (three is sufficient here) and solve the Chinese Reminder Theorem. The three is sufficinent because the message is short (w.r.t. to moduli), for longer message, you will need more queries.

```
from Crypto.Util.number import long_to_bytes
import libnum

# Add the cyphertext and moduli from public key for each message here
c1 = ""
n1 = ""

c2 = ""
n2 = ""

c3 = ""
n3 = ""

e = 5

mod=[int(n1, 16),int(n2, 16),int(n3, 16)]
rem=[int(c1, 16),int(c2, 16),int(c3, 16)]

# solve Chinese Reminder Theorem
res=libnum.solve_crt(rem,mod)
# get e-th root
val=libnum.nroot(res,e)

print(long_to_bytes(val))
```
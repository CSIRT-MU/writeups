# Crypto - xorxorxor

Looking at the code, I can see that the entire thing is using only XORs. However, a particular part pokes my attention:

```
	xored = b''
	for i in range(len(data)):
		xored += bytes([data[i] ^ self.key[i % len(self.key)]])
	return xored
```

This means that the key is just repeated over and over again. That leads to one-time pad with repeated key. And since `ciphertext1 XOR ciphertext2 = plaintext1 XOR plaintext2`, I just need to correctly slice the input. From `self.key = os.urandom(4)` I know that the key is 4 bytes long, so I slice the input accorgingly:

```
134af6e1 297bc4a9 6f6a87fe 046684e8 047084ee 046d84c5 282dd7ef 292dc9
```

Now, to get a key I need some crib. Luckily, I know how the flag stars `HTB{` (that is four bytes)

```
134af6e1 # Ciphertext
4854427b # Plaintext crib "HTB{"
--------
5b1eb49a # Key
```

Now, I just repeat the key (modulo the padding) and XOR the whole thing

```
134af6e1 297bc4a9 6f6a87fe 046684e8 047084ee 046d84c5 282dd7ef 292dc9
5b1eb49a 5b1eb49a 5b1eb49a 5b1eb49a 5b1eb49a 5b1eb49a 5b1eb49a 5b1eb4
---------------------------------------------------------------------
4854427b 72657033 34743364 5f783072 5f6e3074 5f73305f 73336375 72337d
```

And that is the flag!
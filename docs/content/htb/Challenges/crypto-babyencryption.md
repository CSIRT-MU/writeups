---
authors:
    - Lukas Daubner
date: 16-01-2024
---

# Crypto - BabyEncryption

The encryption does not use any secret information. Just modular aritmetirc, which makes it tricky to reverse. That makes it a substitutional cypher.

But since there is no secret, you can precompute the encrypted values for each character (see: `buildDictionary()`). NOTE: I tried to use only the printable characters, but that is a catch. There is a '\n' in there

then you can read it and lookup the plaintext character for a cyperthext character.

```python
# from chall.py
def encryption(msg):
    ct = []
    for char in msg:
        ct.append((123 * char + 18) % 256)
    return bytes(ct)

def buildDictionary():
    dict = {}
    for char in range(256):
        charMsg = [char]
        crypt = int.from_bytes(encryption(charMsg), byteorder='little')
        dict[crypt] = char
    return dict

def decryption(enc):
    dict = buildDictionary()
    pt = []
    for byte in bytes.fromhex(enc):
        p = chr(dict[byte])
        pt.append(p)
    return ''.join(pt)

with open('./msg.enc','r') as f:
    enc = f.read()
    plain = decryption(enc)
    print(plain)
```
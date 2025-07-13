# TryHackMe Room: w1seguy - Writeup

## Overview

**Room:** w1seguy
**Platform:** TryHackMe
**Category:** Crypto, Reverse Engineering
**Difficulty:** Easy-Medium

---
![w1seguy](https://tryhackme-images.s3.amazonaws.com/room-icons/9a13d8dcbd8940d14352ba2edbf66735.png)
---

In this room, we exploit a weak XOR encryption used to protect flags during a simple socket-based challenge. The encryption uses a 5-byte repeating key to XOR a flag that starts with a predictable pattern (`THM{`) and ends with `}`. We will recover the key and decrypt the flag automatically, then submit the key to obtain the second flag.

---

# Examining the Source Code

At the start of the room, we are given the following **Python source code** for the application running on port 1337:

```python
import random
import socketserver
import socket, os
import string

flag = open('flag.txt','r').read().strip()

def send_message(server, message):
    enc = message.encode()
    server.send(enc)

def setup(server, key):
    flag = 'THM{thisisafakeflag}'
    xored = ""
    for i in range(0,len(flag)):
        xored += chr(ord(flag[i]) ^ ord(key[i%len(key)]))
    hex_encoded = xored.encode().hex()
    return hex_encoded

def start(server):
    res = ''.join(random.choices(string.ascii_letters + string.digits, k=5))
    key = str(res)
    hex_encoded = setup(server, key)
    send_message(server, "This XOR encoded text has flag 1: " + hex_encoded + "\n")
    
    send_message(server,"What is the encryption key? ")
    key_answer = server.recv(4096).decode().strip()

    try:
        if key_answer == key:
            send_message(server, "Congrats! That is the correct key! Here is flag 2: " + flag + "\n")
            server.close()
        else:
            send_message(server, 'Close but no cigar' + "\n")
            server.close()
    except:
        send_message(server, "Something went wrong. Please try again. :)\n")
        server.close()

class RequestHandler(socketserver.BaseRequestHandler):
    def handle(self):
        start(self.request)

if __name__ == '__main__':
    socketserver.ThreadingTCPServer.allow_reuse_address = True
    server = socketserver.ThreadingTCPServer(('0.0.0.0', 1337), RequestHandler)
    server.serve_forever()
```

This code explains how the server generates a **random 5-character key**, uses it to XOR encrypt the flag, sends the encrypted hex to the client, and expects the correct key to reveal **flag 2**.

---

## Challenge Recap

* The server sends a hex-encoded XOR ciphertext:

  ```
  This XOR encoded text has flag 1: <hex_encoded>
  ```

* You must:

  * Decrypt it to retrieve **flag 1**.
  * Submit the recovered encryption key to the server.
  * Retrieve **flag 2**.

---

## Technical Analysis

* The encryption uses a **5-byte key repeatedly**.
* Flag format is `THM{...}` which provides a **known-plaintext for the first 4 bytes** and `}` for the last byte.
* Since ciphertext length is divisible by 5, the last ciphertext byte is encrypted with `key[4]`.
* We recover the key by XORing:

  * `ciphertext[0] ^ ord('T')`
  * `ciphertext[1] ^ ord('H')`
  * `ciphertext[2] ^ ord('M')`
  * `ciphertext[3] ^ ord('{')`
  * `ciphertext[-1] ^ ord('}')`

This immediately reveals the encryption key.

Once we recover the key, we can decrypt the entire ciphertext and submit the key back to the server.

---

## Automated Exploit Script

We developed a Python script that:

1. Connects to the server.
2. Receives the hex-encoded ciphertext.
3. Recovers the 5-byte encryption key.
4. Decrypts and prints **flag 1** for your notes.
5. Submits the key back on the same connection.
6. Receives and prints **flag 2** for submission.

---

### Final `crack.py`

```python
import socket

def recover_key(ciphertext_hex):
    ciphertext = bytes.fromhex(ciphertext_hex)
    plaintext = [ord('T'), ord('H'), ord('M'), ord('{')]
    last_plaintext = ord('}')
    key = [0] * 5

    for i in range(4):
        key[i] = ciphertext[i] ^ plaintext[i]
    key[4] = ciphertext[-1] ^ last_plaintext

    return bytes(key)

def decrypt_flag(ciphertext_hex, key_bytes):
    ciphertext = bytes.fromhex(ciphertext_hex)
    decrypted = []

    for i in range(len(ciphertext)):
        decrypted_byte = ciphertext[i] ^ key_bytes[i % 5]
        decrypted.append(chr(decrypted_byte))

    return ''.join(decrypted)

if __name__ == '__main__':
    HOST = 'MACHINE_IP'  # replace with your target IP
    PORT = 1337

    s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    s.connect((HOST, PORT))

    data = s.recv(4096).decode()
    print(data)

    ciphertext_hex = data.split('flag 1: ')[1].split('\n')[0].strip()
    print("[*] Received ciphertext hex:", ciphertext_hex)

    key_bytes = recover_key(ciphertext_hex)
    key_str = key_bytes.decode(errors='ignore')
    print("[*] Recovered key:", key_str)

    flag1 = decrypt_flag(ciphertext_hex, key_bytes)
    print("[*] Decrypted flag 1:", flag1)

    prompt = s.recv(4096).decode()
    print(prompt)

    s.sendall((key_str + '\n').encode())

    flag2 = s.recv(4096).decode()
    print(flag2)

    s.close()
```

---

## Flags

![flags](assets/scr_w1seguy.png)

✅ **Flag 1:** Decrypted locally using your script.
✅ **Flag 2:** Retrieved automatically after submitting the recovered key.

---

## Conclusion

The **w1seguy** TryHackMe room demonstrates how predictable structures in CTF flags combined with weak XOR encryption can be exploited using a systematic approach to recover flags easily. The process builds confidence for attacking crypto challenges in real CTF environments.

Happy hacking! 🚩

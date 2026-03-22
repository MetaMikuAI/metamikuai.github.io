---
title: Padding Oracle Attack学习笔记
date: 2024-03-19 21:54:45
tags:
---

# Padding Oracle Attack学习笔记

## 前言

周末的校赛出了这题，最近一直对数学性强的加密研究较多，对于此类算法性强的加密方法没有过多关注，结果比赛就是被`socket`的通信坑了大半天v_v

这篇文章主要是简单记录一下，具体原理和攻击方法有很多师傅讲的要比我好很多。

而且写完我感觉也挺乱的，似乎只有脚本可以供各位参考

## AEC-CBC加密

加解密基本原理图：

![CBC_encryption.svg](/image/articles/Padding-Oracle-Attack学习笔记.assets/CBC_encryption.svg.jpg)

![1280px-CBC_decryption.svg](/image/articles/Padding-Oracle-Attack学习笔记.assets/1280px-CBC_decryption.svg.jpg)

`CBC`加密时按`AES.block_size`字节分块将明文分块，每块异或分别使用`key`加密，加密后输出同时自身参与下一块的异或操作。解密时每一块密文通过`key`作用再与前一块密文进行异或得到明文。

注意无论是加密还是解密，都与且仅与上一块通过单次异或作用。

## PKCS7 Padding

注意到上述`CBC`加解密过程中将明文密文按`AES.block_size`字节分块，如果明文长度不是`AES.block_size`字节的整数倍呢？需要使用`PKCS7 Padding`的数据填充规则，以`AES.block_size = 16`为例，下面演示`padding`的过程。

无论明文多长，最后一块的长度一定是$\{1,2,3,\cdots,14,15,16\}$，对每种情况的最后一块数据进行填充(包括最后一节为16字节的情况)，补充的字节数是 到下一个16字节倍数所需字节数，每个填充字节的值也是到下一个16字节倍数所需字节数。

听起来很拗口，下面看具体情况：

最后一节为$15$个字节，需要补充$1$个字节达到$16$的倍数，则填充$1$个`0x01`

最后一节为$10$个字节，需要补充$6$个字节达到$16$的倍数，则填充$6$个`0x06`

最后一节为$2$个字节，需要补充$14$个字节达到$16$的倍数，则填充$14$个`0x0e`

最后一节为$16$个字节，需要补充$16$个字节达到$16$的倍数，则填充$16$个`0x10`

因此，补充后最后一节的可能性有如下16种情况：

```
... | XX XX XX XX XX XX XX XX XX XX XX XX XX XX XX 01
... | XX XX XX XX XX XX XX XX XX XX XX XX XX XX 02 02
... | XX XX XX XX XX XX XX XX XX XX XX XX XX 03 03 03
... | XX XX XX XX XX XX XX XX XX XX XX XX 04 04 04 04
... | XX XX XX XX XX XX XX XX XX XX XX 05 05 05 05 05
... | XX XX XX XX XX XX XX XX XX XX 06 06 06 06 06 06
... | XX XX XX XX XX XX XX XX XX 07 07 07 07 07 07 07
... | XX XX XX XX XX XX XX XX 08 08 08 08 08 08 08 08
... | XX XX XX XX XX XX XX 09 09 09 09 09 09 09 09 09
... | XX XX XX XX XX XX 0a 0a 0a 0a 0a 0a 0a 0a 0a 0a
... | XX XX XX XX XX 0b 0b 0b 0b 0b 0b 0b 0b 0b 0b 0b
... | XX XX XX XX 0c 0c 0c 0c 0c 0c 0c 0c 0c 0c 0c 0c
... | XX XX XX 0d 0d 0d 0d 0d 0d 0d 0d 0d 0d 0d 0d 0d
... | XX XX 0e 0e 0e 0e 0e 0e 0e 0e 0e 0e 0e 0e 0e 0e
... | XX 0f 0f 0f 0f 0f 0f 0f 0f 0f 0f 0f 0f 0f 0f 0f
... | 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10
```

下面这种具体情况是**正确**的：

```
35 70 f2 45 09 d3 a3 6b 90 3e 3a 58 03 03 03 03
```

下面这种具体情况是**错误**的：

```
35 70 f2 45 09 d3 a3 6b 90 3e 3a 58 07 02 03 03
```

正确的情况在`Unpadding`中可以正确还原出明文

错误的情况在`Unpadding`中会出错

如果恰巧，当服务端把是否`Unpadding`成功的信息泄露出来，就可以针对其进行破解

## 例题

```python
from base64 import *
from Crypto.Random import get_random_bytes
from Crypto.Cipher import AES
from Crypto.Util.number import *
import os
import string
from hashlib import sha256
import socketserver
import signal
import random
# flag = os.getenv('FLAG').encode()

AES.block_size=16
iv = get_random_bytes(AES.block_size)
key = get_random_bytes(16)

class Task(socketserver.BaseRequestHandler):
    def proof_of_work(self):
        random.seed(os.urandom(8))
        proof = ''.join(
            [random.choice(string.ascii_letters+string.digits) for _ in range(20)])
        _hexdigest = sha256(proof.encode()).hexdigest()
        self.send(f"[+] sha256(XXXX+{proof[4:]}) == {_hexdigest}".encode())
        x = self.recv(prompt=b'[+] Plz tell me XXXX: ')
        if len(x) != 4 or sha256(x+proof[4:].encode()).hexdigest() != _hexdigest:
            return True
        return True

    def _recvall(self):
        BUFF_SIZE = 2048
        data = b''
        while True:
            part = self.request.recv(BUFF_SIZE)
            data += part
            if len(part) < BUFF_SIZE:
                break
        print("[+] "+ data.strip().decode())
        return data.strip()

    def send(self, msg, newline=True):
        try:
            if newline:
                msg += b'\n'
            self.request.sendall(msg)
            print('[-] '+msg.decode())
        except:
            pass

    def recv(self, prompt=b'> '):
        self.send(prompt, newline=False)
        return self._recvall()

    def timeout_handler(self, signum, frame):
        raise TimeoutError


class ThreadedServer(socketserver.ThreadingMixIn, socketserver.TCPServer):
    pass


def pad(text):
    pad_num = 16 - len(text) % 16
    return text + pad_num*bytes([pad_num])


def unpad(text):
    pad_num = int(text[-1])
    if pad_num == 0:
        return b'Man'
    for i in range(1, pad_num+1):
        if int(text[-i]) != pad_num:
            return b'Man'
    else:
        tmp = text[:-pad_num]
        return b'Mamba out!'


def encrypt(plain_text, key):
    cipher = AES.new(key, AES.MODE_CBC, iv)
    cipher_text = cipher.encrypt(pad(plain_text))
    return iv + cipher_text


def decrypt(cipher_text, key):
    iv = cipher_text[:AES.block_size]
    cipher = AES.new(key, AES.MODE_CBC, iv)
    tmp = cipher.decrypt(cipher_text[AES.block_size:])
    result = unpad(tmp)
    return result


class test(Task):
    def handle(self):
        if not self.proof_of_work():
            self.send(b'[!] Wrong!')
            return
        enc = encrypt(flag, key)
        self.send(b'secret:')
        self.send(b64encode(enc))
        self.send(b'What can I say?')
        while True:
            data = self.recv()
            try:
                self.send(decrypt(b64decode(data), key))
            except:
                self.send(b'Damn!')


if __name__ == "__main__":
    HOST, PORT = '0.0.0.0', 4444
    server = ThreadedServer((HOST, PORT), test)
    server.allow_reuse_address = True
    print(HOST, PORT)
    server.serve_forever()
```

首先进工作量证明，通过后得到明文加密后的(base64)信息，然后用户即可发送数据让服务端进行验证。

## 攻击流程

攻击流程也可以大致写出：

1. 保存真正明文加密后的密文，其中前16字节为`IV`初始化向量
2. 创建一个$32$字节的数组，用于实时更新将要攻击的数据列表，前$16$字节位初始化为先前得到的`IV`向量；创建一个$16$字节的数组，用于寄存中间值。
3. 对攻击列表的最后一位从`0x00`到`0x01`遍历，直到服务端的`Unpadding`成功，且当改变倒数第二位时，`Unpadding`过程仍然通过，证明此轮`Unpadding`的结果末位确实为`0x01`
4. 把攻击列表的末位与`0x01`异或得到的结果即为此轮解密后的中间值
5. 把攻击列表的末位修改为中间值 异或 `0x02`以实施对倒数第二位的攻击
6. 从第三步重复，对倒数第二位实施攻击，直到$16$个字节均攻击完毕
7. 改攻击列表前$16$字节为密文第一组，从第三步重复，直到每部分密文攻击完毕

## 攻击参考脚本

```python
from base64 import *
from pwn import *

# 工作量证明
def crack_proof_of_work(proof4:str,_hexdigest:str) -> bytes:
    from hashlib import sha256
    print("Task get: ",proof4,_hexdigest)
    import string
    alphabet = string.ascii_letters+string.digits
    for a in alphabet:
        for b in alphabet:
            for c in alphabet:
                for d in alphabet:
                    if sha256((a+b+c+d+proof4).encode()).hexdigest() == _hexdigest:
                        print(f"proof of work cracked: {a}{b}{c}{d}")
                        return (a+b+c+d).encode()

# 连接远程主机
s = remote("127.0.0.1", 4444)
proof_of_work_problem = s.recvline(keepends=False).decode()
print(s.recv()) # b'[+] Plz tell me XXXX: '
proof_of_work_answer = crack_proof_of_work(proof_of_work_problem[16:32],proof_of_work_problem[37:101])
s.sendline(proof_of_work_answer)
print(s.recvline(keepends=True)) # b'secret:\n'
encbase64bytes = s.recvline(keepends=False)
print(s.recv()) # b'What can I say?\n> '

flag = ''

for N in [0,16,32]: # 原题密文去掉IV向量就只有48字节分3段，按实际情况更改
    middle_list = [0,0,0,0, 0,0,0,0, 0,0,0,0, 0,0,0,0]
    attack_list = [0,0,0,0, 0,0,0,0, 0,0,0,0, 0,0,0,0]
    attack_list.extend(list(b64decode(encbase64bytes))[16+N:32+N])
    s.send(b64encode(bytes(attack_list)))
    for index in range(15,-1,-1):
        for i in range(256):
            attack_list[index] = i
            s.send(b64encode(bytes(attack_list)))
            res = s.recvline(keepends=False)
            # print(res)
            _ = s.recv()
            if res != b'Man':
                break
        if index == 0:
            print(f"Got one bit! {index}",end=' \r')
            middle_list[index] = attack_list[index] ^ (16-index)
            break
        attack_list[index-1] ^= 0x0f # 反转比特 验证
        s.send(b64encode(bytes(attack_list)))
        res = s.recvline(keepends=False)
        # print(res)
        _ = s.recv()
        if res != b'Man':
            print(f"Got one bit! {index}",end='\r')
            middle_list[index] = attack_list[index] ^ (16-index)
            # 初始化下一padding
            for j in range(index,16):
                attack_list[j] = middle_list[j] ^ (16-index+1)
            index -= 1
        else:
            # 没想好怎么写这个分支，反正一般用不着吧（
            pass
    flag_ = bytes([middle_list[i] ^ list(b64decode(encbase64bytes))[0+N:16+N][i] for i in range(16)]).decode()
    flag += flag_
    print(f"\nflag got: {flag_} \n {flag}")

```

## 结语

我真的感觉文章写完后超级超级乱，就留着自用吧（逃

## 参考学习

- 非常清晰的动画讲解↓
[Padding Oracle 攻击 - BiliBili](https://www.bilibili.com/video/av828688849)
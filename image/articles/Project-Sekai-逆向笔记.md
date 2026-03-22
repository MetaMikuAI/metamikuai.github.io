---
title: Project Sekai 逆向笔记
date: 2025-04-01 22:04:31
tags:
---

# Project SEKAI 逆向笔记 持续更新

非常喜欢 Project Sekai 这款游戏，不过我本身是主要学习密码学的，对逆向不甚了解，仅向下面博客进行了拙劣的学习，尝试提供一些基础的实践尝试，可能会记录得过于详细。

↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓

pjsk逆向唯一神贴: [Project SEKAI 逆向 - 笔记归档](https://mos9527.com/posts/pjsk/archive-20240105/)

↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑↑

## 更新日志

- `20250410` 完成了 `静态解包` `第三方工具` 等部分的大致整理
- `20250411` 完成了 `提取 global-metadata.dat` 部分的整理

## 静态解包

参见：[【世界计划 多彩舞台】PJSK游戏资源解包教程-补充资料：新版](https://www.bilibili.com/opus/579098944556614370)

## 动态分析

### 提取 `global-metadata.dat` (成功)

参考 [Project SEKAI 逆向 - 笔记归档](https://mos9527.com/posts/pjsk/archive-20240105/)

研究版本：Project Sekai 国服 3.4.0

#### 游戏安装包(失败)

拿到公测开始公布的安装包，解压，找到`./assets/bin/Data/Managed/Metadata/global-metadata.dat`

发现 `AF 1B B1 FA`，但是在第九个字节开始，真的没加密吗？*~~我不信会这么仁慈~~*

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411153730114.png)

尝试把前八个字节删掉，并找到 `./lib/arm64-v8a/libil2cpp.so`，一并丢给 [Il2CppDumper](https://github.com/Perfare/Il2CppDumper/)

果不其然报错了，而且是运行一半之后才报错，如图

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411154451449.png)

查看 `hexdump` 发现整个文件到处都是乱码，应该还是有加密

#### `/proc/[PID]/maps` 提取 `libcamera-metadata.dat` (失败)

参照 [metadata 动态提取](https://mos9527.com/posts/pjsk/archive-20240105/#1-metadata-%E5%8A%A8%E6%80%81%E6%8F%90%E5%8F%96) 中的方法

启动游戏，进 `adb shell` 用 `top` 命令拿 `PID=31607`

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411155027155.png)

`su` 后在 `/proc/[PID]/maps` 查找 `metadata`，结果如图

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411155233392.png)

使用 `dd` 来 `dump` 这三段内存

```shell
dd if=/proc/31607/mem of=/sdcard/libcamera_metadata.so.1 bs=1 skip=$(printf "%u" 0x715074d000) count=$((0x7150755000-0x715074d000))
dd if=/proc/31607/mem of=/sdcard/libcamera_metadata.so.2 bs=1 skip=$(printf "%u" 0x715076a000) count=$((0x715076b000-0x715076a000))
dd if=/proc/31607/mem of=/sdcard/libcamera_metadata.so.3 bs=1 skip=$(printf "%u" 0x715076b000) count=$((0x715076d000-0x715076b000))
```

第一个是 `ELF`，第二个第三个不知道是什么东西

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411165850349.png)

#### 内存搜索 `AF 1B B1 FA` (失败)

分别使用 Frida 和 GameGuardian 使用众多方法在内存中搜索魔术头 `AF 1B B1 FA` 无果。

注意：使用 GameGuardian 有封号风险，谨慎使用

![](/image/articles/Project-Sekai-逆向笔记.assets/被朝夕ban了.jpg)

痛失小号×1

~~*其实还可能是因为出于恶搞，用 GG 在游戏里面改了 2147483647 个水晶（当然只是前端）*~~

![](/image/articles/Project-Sekai-逆向笔记.assets/2147483647.jpg)

当时 root 机截图坏了，临时拍了个屏

#### 内存搜索特殊字段(成功)

虽然被 ban 了，但是毕竟游戏的 metadata 已经被加载了，尝试继续在游戏开始界面

还是在 [metadata 动态提取](https://mos9527.com/posts/pjsk/archive-20240105/#1-metadata-%E5%8A%A8%E6%80%81%E6%8F%90%E5%8F%96) 中，发现其他服中的 metadata 有这样的字符串

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411171057733.png)

GG 搜了一下 `mscorlib`，有好多结果，又搜了一下 `mscorlib.dll.<Module>`(注意用十六进制搜，里面有的点并不是字符点)，我草，唯一结果

![](/image/articles/Project-Sekai-逆向笔记.assets/GG_1.png)

所在内存地址 `0x7B26457032` ，顺便看一下内存上下文，确实有像 metadata 的字符串

到 `/proc/[PID]/maps` 找这段内存

```shell
cat /proc/4958/maps | more
```

```shell
7b25f88000-7b25f89000 ---p 00000000 00:00 0                              [anon:thread stack guard]
7b25f89000-7b25f8a000 ---p 00000000 00:00 0
7b25f8a000-7b2608e000 rw-p 00000000 00:00 0
7b2629a000-7b2629b000 ---p 00000000 00:00 0                              [anon:thread stack guard]
7b2629b000-7b2629c000 ---p 00000000 00:00 0
7b2629c000-7b263a0000 rw-p 00000000 00:00 0
7b263a0000-7b2761f000 rw-p 00000000 00:00 0
7b2761f000-7b27e08000 rw-s 00000000 00:0b 9591                           anon_inode:dmabuf
7b27e08000-7b2f500000 r-xp 00000000 b3:42 93921                          /data/app/com.hermes.mk-N1TZmXwSQOMJzh2TegB6vA==/lib/arm64/libil2cpp.so
--More--
```

找到

```shell
7b263a0000-7b2761f000 rw-p 00000000 00:00 0
```

用 `dd` dump 出这段内存

```shell
dd if=/proc/4958/mem of=/sdcard/dump.dat bs=1 skip=$(printf "%u" 0x7b263a0000) count=$((0x7b2761f000-0x7b263a0000))
```

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411192917347.png)

一个很像 `global-metadata.dat` 的文件，在 `WinMerge` 中严谨对比一下

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411193300514.png)

经过比对，相较于 apk 中解包出的 `metadata`，dump 出的 `metadata` 具有以下五点不同

1. 魔数 `AF 1B B1 FA` 被改为 `00 00 00 00`，这是先前全局搜索内存无果的原因*~~（朝夕你真毒啊）~~*
2. 魔数后的第一个字节 `1B` 被改为了 `18`，原因未知
3. `D0 39 1E 00` -> `FF FF FF 7F`，事后猜测这似乎是一个被改到`INT_MAX`的 `int` 值
4. 加密段被解密，事后经验证为一段异或加密
5. 尾部多出一堆 `00`，大概是分配内存多余的空间

结合两个 `metadata`，经过多次尝试，最终需要还原的 `global-metadata.dat` 方法如下：

1. 取内存 dump 出的 `metadata`
2. 比对 apk 中解包出的 `metadata`，删除多余的 `00` 字节
3. 复原 `FF FF FF 7F` -> `D0 39 1E 00`
4. 复原 `18` -> `1B`
5. 复原 `00 00 00 00` -> `AF 1B B1 FA`
6. 删除头 `8` 字节，即使 `AF 1B B1 FA` 位于文件开头

最后使用工具 [Il2CppDumper](https://github.com/Perfare/Il2CppDumper/) 解析 `libil2cpp.so`

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411223143942.png)

使用 `dnSpy` 反编译 `./DummyDll/Assembly-CSharp.dll`

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411223309868.png)

有 [Beebyte](https://www.beebyte.co.uk/) 但是类名函数名没混淆

任务完成

恢复好的 `global-metadata.dat` 的 `MD5` 为 `9069c4db8cd9c520c2d22be69c36d30b`

#### 静态分析 `libil2cpp.so` (失败)

已经通过 `dump` 和修补拿到了完整的 `global-metadata.dat`，但是**没有**成功通过逆向 `libil2cpp.so` 发现解密算法，下面分享一下我的思路思路

参考文章: 

1. [解密 metadata](https://mos9527.com/posts/pjsk/archive-20240105/#1-%E8%A7%A3%E5%AF%86-metadata)
2. [某手游il2cpp逆向分析----libtprt保护](https://www.52pojie.cn/thread-2010789-1-1.html)

使用 `IDA` 打开 `./lib/arm64-v8a/libil2cpp.so`，直接 `Shift+F12` 搜 `metadata`

![image-20250416002147624](/image/articles/Project-Sekai-逆向笔记.assets/image-20250416002147624.png)

*(注：部分变量函数已更名，但我基本不会逆向，可能有误标)*

![image-20250416002305548](/image/articles/Project-Sekai-逆向笔记.assets/image-20250416002305548.png)

进 `LoadMetadataFile`

```c
_BYTE *__fastcall LoadMetadataFile(char *string_global_metadata_dat)
{
  void *v2; // x8
  char *v3; // x9
  __int64 strlen_global_metadata_dat; // x0
  char *v5; // x8
  void *v6; // x9
  __int64 v7; // x0
  const char *v8; // x1
  __int64 v9; // x20
  _BYTE *metadata_mapped; // x19
  char *v12; // [xsp+0h] [xbp-60h] BYREF
  __int64 v13; // [xsp+8h] [xbp-58h]
  void *v14[2]; // [xsp+10h] [xbp-50h] BYREF
  void *p; // [xsp+20h] [xbp-40h]
  void *v16[2]; // [xsp+28h] [xbp-38h] BYREF
  void *v17; // [xsp+38h] [xbp-28h]
  char *error; // [xsp+40h] [xbp-20h] BYREF
  void *v19; // [xsp+48h] [xbp-18h]

  sub_1915B10(v14);
  v12 = "Metadata";
  v13 = 8LL;
  v2 = (void *)((unsigned __int64)LOBYTE(v14[0]) >> 1);
  if ( ((__int64)v14[0] & 1) != 0 )
    v3 = (char *)p;
  else
    v3 = (char *)v14 + 1;
  if ( ((__int64)v14[0] & 1) != 0 )
    v2 = v14[1];
  error = v3;
  v19 = v2;
  sub_189CEEC((__int64)&error, (__int64)&v12, v16);
  if ( ((__int64)v14[0] & 1) != 0 )
    operator delete(p);
  strlen_global_metadata_dat = strlen(string_global_metadata_dat);
  if ( ((__int64)v16[0] & 1) != 0 )
    v5 = (char *)v17;
  else
    v5 = (char *)v16 + 1;
  if ( ((__int64)v16[0] & 1) != 0 )
    v6 = v16[1];
  else
    v6 = (void *)((unsigned __int64)LOBYTE(v16[0]) >> 1);
  v12 = string_global_metadata_dat;
  v13 = strlen_global_metadata_dat;
  error = v5;
  v19 = v6;
  sub_189CEEC((__int64)&error, (__int64)&v12, v14);
  LODWORD(error) = 0;
  v7 = osFileOpen((__int64)v14, 3, 1u, 1u, 0, &error);
  if ( (_DWORD)error )
  {
    if ( ((__int64)v14[0] & 1) != 0 )
      v8 = (const char *)p;
    else
      v8 = (char *)v14 + 1;
    sub_1915180("ERROR: Could not open %s", v8);
  }
  else
  {
    v9 = v7;
    metadata_mapped = (_BYTE *)utilsMemoryMappedFileMap(v7);
    sub_1892FFC(v9, &error);
    if ( !(_DWORD)error )
      goto LABEL_22;
    sub_191534C(metadata_mapped);
  }
  metadata_mapped = 0LL;
LABEL_22:
  if ( ((__int64)v14[0] & 1) != 0 )
    operator delete(p);
  if ( ((__int64)v16[0] & 1) != 0 )
    operator delete(v17);
  return metadata_mapped;
}
```

对比 `Unity` 的默认实现

```c++
void* il2cpp::vm::MetadataLoader::LoadMetadataFile(const char* fileName)
{
    std::string resourcesDirectory = utils::PathUtils::Combine(utils::Runtime::GetDataDir(), utils::StringView<char>("Metadata"));

    std::string resourceFilePath = utils::PathUtils::Combine(resourcesDirectory, utils::StringView<char>(fileName, strlen(fileName)));

    int error = 0;
    os::FileHandle* handle = os::File::Open(resourceFilePath, kFileModeOpen, kFileAccessRead, kFileShareRead, kFileOptionsNone, &error);
    if (error != 0){
        utils::Logging::Write("ERROR: Could not open %s", resourceFilePath.c_str());
        return NULL;
    }

    void* fileBuffer = utils::MemoryMappedFile::Map(handle);

    os::File::Close(handle, &error);
    if (error != 0)
    {
        utils::MemoryMappedFile::Unmap(fileBuffer);
        fileBuffer = NULL;
        return NULL;
    }

    return fileBuffer;
}
```

这段代码的特征和文章 [某手游il2cpp逆向分析----libtprt保护](https://www.52pojie.cn/thread-2010789-1-1.html) 中的例子极其相似，对照进行分析

`v7 = osFileOpen((__int64)v14, 3, 1u, 1u, 0, &error);` 读取文件后紧跟着异常处理，然后进 `utilsMemoryMappedFileMap`

### 获取 AES key (成功)

配置工具链 root 物理机 + frida + frida-il2cpp-bridge

hook 下面脚本

脚本来自 [Project SEKAI 逆向 - 笔记归档 - IL2CPP 动态调用 runtime](https://mos9527.com/posts/pjsk/archive-20240105/#3-il2cpp-%E5%8A%A8%E6%80%81%E8%B0%83%E7%94%A8-runtime)

```typescript
import "frida-il2cpp-bridge";

declare namespace console {
    function log(...args: any[]): void;
}

Il2Cpp.perform(() => {

    Il2Cpp.domain.assemblies.forEach((i)=>{
        console.log(i.name);
    })

    console.log("started")
    const game = Il2Cpp.domain.assembly("Assembly-CSharp").image;
    const apiManager = game.class("Sekai.APIManager");
    Il2Cpp.gc.choose(apiManager).forEach((instance: Il2Cpp.Object) => {
        console.log("instance found")
        const crypt = instance.method<Il2Cpp.Object>("get_Crypt").invoke();
        const aes = crypt.field<Il2Cpp.Object>("aesAlgo");
        const key = aes.value.method("get_Key").invoke();
        const iv = aes.value.method("get_IV").invoke();
        console.log(key);
        console.log(iv);
    });
});
```

成功捕获 `key` 和 `iv`

```python
# key = [103,50,102,99,67,48,90,99,122,78,57,77,84,74,54,49]
# iv = [109,115,120,51,73,86,48,105,57,88,69,53,117,89,90,49]

key = b'g2fcC0ZczN9MTJ61'
iv = b'msx3IV0i9XE5uYZ1'
```

## API 分析

### MITM

工具：[mitmproxy](https://www.mitmproxy.org/) [WireGurad](https://www.wireguard.com/) [MuMu模拟器](https://mumu.163.com/)

可以参阅详细教程 [Mitmproxy方案使用教程 ](https://github.com/BanterSR/ci/blob/main/BaPs/Android_Mitmproxy.md)也可以参考下面简略教程

1. 在 PC 上安装 mitmproxy
2. 在电脑端安装证书 `C:\Users\用户\.mitmproxy\mitmproxy-ca.p12` 
3. 将 `C:\Users\用户\.mitmproxy\mitmproxy-ca-cert.cer` 重命名为 `c8750f0d.0`
4. 将 `c8750f0d.0` 移动到安卓设备的 `/system/etc/security/cacerts/c8750f0d.0` 或通过模块安装到系统 CA
5. 电脑端运行 `mitmweb -m wireguard --no-http2 -s 脚本.py --set termlog_verbosity=warn --ignore 这里输入你的IP地址` 启动 mitmproxy
6. 安卓端启动 wireguard 并扫码连接

至此可以形成这样的代理链

```
正常情况下
MobileDevice  -------->  -------->  -------->  --------> --------> The Server
                           - - ->X  mitmproxy (but not trut)

使用 MITM                  
MobileDevice  -------->  --------> mitmproxy  - - - - >  - - - - > The Server
(With MITM CA)                         | 
                                       |  (if redirection rule is set)
                                       v  
                                Hacker's Server
```

### API URLs

抓包发现主要有如下几个地址 (My Sekai 烤森除外，这个好像是特殊的 api)

```
https://production-game-api.sekai.colorfulpalette.org -> 最最最主要的 api, GET POST PUT 都有
https://game-version.sekai.colorfulpalette.org -> 只在启动时请求, 返回字段包含 profile assetbundleHostHash domain, 其中 domain 总为 https://production-game-api.sekai.colorfulpalette.org
https://issue.sekai.colorfulpalette.org -> 并不常见, 似乎只在有问题的时候才会
https://cdp.cloud.unity3d.com/v1/events -> u3d 收集信息的地址, 不用搭理
```

只要第一个 api 即可实现基本游玩需求

### 分析加密

第一层显而易见是 HTTP，使用 mitmproxy 轻松解决

接下来的分析还是来自 [Project SEKAI 逆向 - 笔记归档](https://mos9527.com/posts/pjsk/archive-20240105/)，先是一层 AES-CBC，密钥和初始化向量在本文的 `动态分析-获取AES key` 中获取

然后是 MessagePack 格式的信息，可以直接解码得到 json

临时分析时可以使用 [CyberChef](https://cyberchef.org/) 对流量包轻松解密

<img src="/image/articles/Project-Sekai-逆向笔记.assets/image-20250914193432369.png" alt="image-20250914193432369" style="zoom:50%;" />

为了便于 mitmproxy 自动化抓包解密和后期可能的 PS 计划，编写 python 加解密脚本如下

```python
import json
import msgpack
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad

# get key and iv from frida-il2cpp-bridge first
key = b'g2fcC0ZczN9MTJ61'
iv = b'msx3IV0i9XE5uYZ1'

def decrypt_aes_cbc(data: bytes) -> bytes:
    cipher = AES.new(key, AES.MODE_CBC, iv)
    decrypted_data = unpad(cipher.decrypt(data), AES.block_size)
    return decrypted_data

def encrypt_aes_cbc(data: bytes) -> bytes:
    cipher = AES.new(key, AES.MODE_CBC, iv)
    padded_data = pad(data, AES.block_size)
    encrypted_data = cipher.encrypt(padded_data)
    return encrypted_data

def encode_msgpack(data: dict) -> bytes:
    if data is None:
        raise ValueError("Input data for encode_msgpack cannot be None")
    packed = msgpack.packb(data, use_bin_type=True)
    if not isinstance(packed, bytes):
        raise TypeError("msgpack.packb did not return bytes")
    return packed

def decode_msgpack(data: bytes) -> dict:
    return msgpack.unpackb(data, raw=False)

def pjsk_encode_msgpack(data: dict) -> bytes:
    packed = encode_msgpack(data)
    encrypted = encrypt_aes_cbc(packed)
    return encrypted

def pjsk_decode_msgpack(data: bytes) -> dict:
    decrypted = decrypt_aes_cbc(data)
    decoded = decode_msgpack(decrypted)
    return decoded
```

**注：除了极个别 request 中是 `16` 字节的 magic payload (在本文的测试环境和测试账号中是 `ffa3bd6214f33fe73cb72fee2262bedb`) 外，其余所有 request/response 的payload 均以该方法加密，本文的剩余部分将直接解密不再重复强调**

### [整活] 十连十彩 (成功)

在正式 API 分析前，整个活

**目的：在*不侵害*原 colorful stage *任何利益*的前提下，获得十连十彩的视频**

观察发现抽卡的时候会向形如 `https://production-game-api.sekai.colorfulpalette.org/api/user/1145141919810/gacha/803/gachaBehaviorId/3139?isPriorityUsePaidJewel=False` 的 API 发送 `request`，并在 `response` 中返回一大堆东西，`response` 的 `obtainPrizes` 键下有十个卡牌信息，便是抽卡结果信息，例如

```json
"obtainPrizes": [
    {
        "card": {
            "resourceId": 1236,
            "resourceType": "card",
            "resourceLevel": 1,
            "quantity": 1
        },
        "newFlg": true,
        "gachaLotteryType": "normal",
        "costume3d": [
            {
                "resourceId": 844137,
                "resourceType": "costume_3d",
                "resourceLevel": 0,
                "quantity": 1
            },
            {
                "resourceId": 844138,
                "resourceType": "costume_3d",
                "resourceLevel": 0,
                "quantity": 1
            },
            {
                "resourceId": 844161,
                "resourceType": "costume_3d",
                "resourceLevel": 0,
                "quantity": 1
            }
        ],
        "cardExtra": []
    },
    {
        "card": {
            "resourceId": 972,
            "resourceType": "card",
            "resourceLevel": 1,
            "quantity": 1
        },
        "newFlg": false,
        "gachaLotteryType": "normal",
        "costume3d": [],
        "cardExtra": []
    },
    {
        "card": {
            "resourceId": 127,
            "resourceType": "card",
            "resourceLevel": 1,
            "quantity": 1
        },
        "newFlg": false,
        "gachaLotteryType": "normal",
        "costume3d": [],
        "cardExtra": []
    },
    {
        "card": {
            "resourceId": 10,
            "resourceType": "card",
            "resourceLevel": 1,
            "quantity": 1
        },
        "newFlg": false,
        "gachaLotteryType": "normal",
        "costume3d": [],
        "cardExtra": []
    },
    {
        "card": {
            "resourceId": 316,
            "resourceType": "card",
            "resourceLevel": 1,
            "quantity": 1
        },
        "newFlg": false,
        "gachaLotteryType": "normal",
        "costume3d": [],
        "cardExtra": []
    },
    {
        "card": {
            "resourceId": 26,
            "resourceType": "card",
            "resourceLevel": 1,
            "quantity": 1
        },
        "newFlg": false,
        "gachaLotteryType": "normal",
        "costume3d": [],
        "cardExtra": []
    },
    {
        "card": {
            "resourceId": 431,
            "resourceType": "card",
            "resourceLevel": 1,
            "quantity": 1
        },
        "newFlg": false,
        "gachaLotteryType": "normal",
        "costume3d": [],
        "cardExtra": []
    },
    {
        "card": {
            "resourceId": 431,
            "resourceType": "card",
            "resourceLevel": 1,
            "quantity": 1
        },
        "newFlg": false,
        "gachaLotteryType": "normal",
        "costume3d": [],
        "cardExtra": []
    },
    {
        "card": {
            "resourceId": 401,
            "resourceType": "card",
            "resourceLevel": 1,
            "quantity": 1
        },
        "newFlg": false,
        "gachaLotteryType": "normal",
        "costume3d": [],
        "cardExtra": []
    },
    {
        "card": {
            "resourceId": 148,
            "resourceType": "card",
            "resourceLevel": 1,
            "quantity": 1
        },
        "newFlg": false,
        "gachaLotteryType": "normal",
        "costume3d": [],
        "cardExtra": []
    }
],
```

`obtainPrizes` 下一共 10 个卡面信息，`resourceId` 为卡面 ID，可在 [SekaiViewer](https://sekai.best/) 快速获取，`newFlg` 指示是否为第一次抽到， `newFlg` 为 `true` 且具有服装的真四星需要附加 `costume3d` 信息

因此思路很简单，截取服务器返回包，改 `resourceId` 为十个四星卡即可

exp 如下

```python
# the more miku the better!
# author: [MetaMiku](https://github.com/MetaMikuAI)

from mitmproxy import http
from Crypto.Cipher import AES
from Crypto.Util.Padding import pad, unpad
import msgpack
import json


# get key and iv from frida-il2cpp-bridge first
key = b'g2fcC0ZczN9MTJ61'
iv = b'msx3IV0i9XE5uYZ1'


target_url_list = [
    "https://production-game-api.sekai.colorfulpalette.org",
]


def decrypt_aes_cbc(data: bytes) -> bytes:
    cipher = AES.new(key, AES.MODE_CBC, iv)
    decrypted_data = unpad(cipher.decrypt(data), AES.block_size)
    return decrypted_data

def encrypt_aes_cbc(data: bytes) -> bytes:
    cipher = AES.new(key, AES.MODE_CBC, iv)
    padded_data = pad(data, AES.block_size)
    encrypted_data = cipher.encrypt(padded_data)
    return encrypted_data


def encode_msgpack(data: dict) -> bytes:
    if data is None:
        raise ValueError("Input data for encode_msgpack cannot be None")
    packed = msgpack.packb(data, use_bin_type=True)
    if not isinstance(packed, bytes):
        raise TypeError("msgpack.packb did not return bytes")
    return packed

def decode_msgpack(data: bytes) -> dict:
    return msgpack.unpackb(data, raw=False)


def dict_to_pretty_json(data: dict) -> str:
    return json.dumps(data, ensure_ascii=False, indent=2)


def pjsk_encrypt_msgpack(data: dict) -> bytes:
    packed = encode_msgpack(data)
    encrypted = encrypt_aes_cbc(packed)
    return encrypted


def pjsk_decrypt_msgpack(data: bytes) -> dict:
    decrypted = decrypt_aes_cbc(data)
    decoded = decode_msgpack(decrypted)
    return decoded


def hack_gacha_results(data: dict) -> dict:
    target_data = {
        "card": {
            "resourceId": 1235,
            "resourceType": "card",
            "resourceLevel": 1,
            "quantity": 1
        },
        "newFlg": False,
        "gachaLotteryType": "normal",
        "costume3d": [],
        "cardExtra": []
    }


    gacha_results = data["obtainPrizes"]
    for idx, result in enumerate(gacha_results):
        id = result["card"]["resourceId"]
        print(f"Gacha ID: {id}")
        gacha_results[idx] = target_data
    
    data["obtainPrizes"] = gacha_results
    return data

def request(flow: http.HTTPFlow) -> None: # type: ignore
    for target_url in target_url_list:
        print(flow.request.pretty_url)
        if flow.request.pretty_url.startswith(target_url):
            pass


def response(flow: http.HTTPFlow) -> None: # type: ignore
    for target_url in target_url_list:
        print(flow.request.pretty_url)
        if flow.request.pretty_url.startswith(target_url):
            if "gachaBehaviorId" in flow.request.pretty_url:
                original_data = pjsk_decrypt_msgpack(flow.response.content)
                modified_data = hack_gacha_results(original_data)
                re_encrypted = pjsk_encrypt_msgpack(modified_data)
                flow.response.content = re_encrypted

# mitmweb -m wireguard --no-http2 -s ./mitmproxy.py --set termlog_verbosity=info --ignore 127.0.0.1

```

**the more Miku the better!**

<img src="/image/articles/Project-Sekai-逆向笔记.assets/the_more_miku_the_better.jpg" alt="more miku more better" style="zoom:50%;" />

### 分析 JWT (基本完成)

1. 观察到流量中有大量 JWT 以 `X-Session-Token` `credential` `sessionToken` `signature` `param`  等处出现
2. 除 `param` 外，所有 jwt 均包含用户的 `userId` 和一个未知含义的 uuid
3. 所有 jwt 的签名密钥均未知 (也有可能是我菜，尝试 frida hook 了一些哈希函数无果，大概本地完全不验签，只有服务器会验签)
4. 所有 jwt 的 header 均为：

```json
{
  "typ": "JWT",
  "alg": "HS256"
}
```

速写了一段用于 JWT 分析（加解密函数略）

```python
from mitmproxy import http

LOG_FILE = "analyze_jwt.txt"

targets = [
    "https://production-game-api.sekai.colorfulpalette.org"
]

def request(flow: http.HTTPFlow) -> None:
    for target in targets:
        if flow.request.pretty_url.startswith(target):
            with open(LOG_FILE, "a") as f:
                f.write(f"Request: {flow.request.method} {flow.request.pretty_url}\n")
                if "X-Session-Token" in flow.request.headers:
                    f.write(f"X-Session-Token: {flow.request.headers['X-Session-Token']}\n")
                if flow.request.content:
                    if len(flow.request.content) == 16:
                        return

                    try:
                        data = pjsk_decode_msgpack(flow.request.content)
                        jwt_fields = detect_jwt_in_json(data)
                        if jwt_fields:
                            f.write("Detected JWT fields in JSON payload:\n")
                            f.write(json.dumps(jwt_fields, indent=2) + "\n")
                    except json.JSONDecodeError:
                        pass
                f.write("\n")

def response(flow: http.HTTPFlow) -> None:
    for target in targets:
        if flow.request.pretty_url.startswith(target):
            with open(LOG_FILE, "a") as f:
                f.write(f"Response: {flow.request.method} {flow.request.pretty_url}\n")
                if "X-Session-Token" in flow.response.headers:
                    f.write(f"X-Session-Token: {flow.response.headers['X-Session-Token']}\n")
                if flow.response.content:
                    if len(flow.request.content) == 16:
                        return
                    try:
                        data = pjsk_decode_msgpack(flow.response.content)
                        jwt_fields = detect_jwt_in_json(data)
                        if jwt_fields:
                            f.write("Detected JWT fields in JSON payload:\n")
                            f.write(json.dumps(jwt_fields, indent=2) + "\n")
                    except json.JSONDecodeError:
                        pass
                f.write("\n\n")

# 毕竟是临时的脚本, 嵌套多了一点就多一点吧 -- MetaMiku

def is_jwt(token: str) -> bool:
    parts = token.split('.')
    return len(parts) == 3 and all(part.isascii() for part in parts) and not parts[0].startswith('http') and len(token) > 32

def detect_jwt_in_json(data: dict) -> dict:
    jwt_fields = {}
    for key, value in data.items():
        if isinstance(value, str) and is_jwt(value):
            jwt_fields[key] = value
        elif isinstance(value, dict):
            nested_jwt = detect_jwt_in_json(value)
            if nested_jwt:
                jwt_fields[key] = nested_jwt
        elif isinstance(value, list):
            for item in value:
                if isinstance(item, dict):
                    nested_jwt = detect_jwt_in_json(item)
                    if nested_jwt:
                        jwt_fields[key] = nested_jwt
    return jwt_fields

```

1. 启动游戏时会发送第一个请求 `GET /api/system` 不携带 `X-Session-Token`
2. 然后访问 `/api/user/<userId>/auth?refreshUpdatedResources=False` 并在 payload 中携带 `credential`，每次登录的时候都会携带该 `credential` 不变 (该 `credential` 与用户绑定，所谓更新丢账号应该就是丢的它)，服务端返回 `sessionToken`
3. 在此之后，客户端的**所有请求**都需携带该 `sessionToken`
4. 服务端除 `/api/system` 和 `/api/information` 外，都需要返回新的 `sessionToken`，客户端收到后更新 `sessionToken`，并在下次请求中携带新的 `sessionToken`（平时网络不好导致卡退到主界面大概就是因为这个不同步了吧）保证了前向安全性
5. 该 `credential` **不随** *chong'xin登录* 或 *设置引继码* 而变，但是会在确认继承引继码时接收服务器分发的**新**的 `credential`
6. `userRegistration` 的 `signature` 似乎永远不变
7. 注：jwt 的签名密钥无从得知，如果需要在官方服务器和个人服务器之间反复测试的话，个人服务器**不得分发**新的 `credential` （但是可以重放），以免账号丢失

### 注册新账号

1. 携带本机设备硬件信息访问 `/api/user`

```json
{
    "platform": "Android",
    "deviceModel": "Samsung SM-S9210",
    "operatingSystem": "Android OS 12 / API-32 (V417IR/1974)"
}
```

服务端将返回一个包括 `userRegistration` `credential` `updatedResources` 三个字段的新用户包

2. 客户端携带 `credential` 访问 `/api/user/<user_id>/auth?refreshUpdatedResources=False` 以请求本次会话的初始 `X-Session-Token` （注：此次请求可能会携带旧账号的 `X-Session-Token`），该包其余部分与新账号无关，注意到其中有 `suiteMasterSplitPath` 字段如

```json
{
    "sessionToken": "脱敏",
    "appVersion": "5.6.0",
    "multiPlayVersion": "miku",
    "dataVersion": "5.6.0.30",
    "assetVersion": "5.6.0.30",
    "removeAssetVersion": "1.13.0.30",
    "assetHash": "5242f35c-e32d-bafb-a488-74b247052713",
    "appVersionStatus": "available",
    "isStreamingVirtualLiveForceOpenUser": false,
    "deviceId": "c8b6d5f1-f96a-4588-b650-c2a7452e4bd6",
    "updatedResources": {},
    "suiteMasterSplitPath": [
        "suitemasterfile/5.6.0.30/00_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41",
        "suitemasterfile/5.6.0.30/01_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41",
        "suitemasterfile/5.6.0.30/02_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41",
        "suitemasterfile/5.6.0.30/03_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41",
        "suitemasterfile/5.6.0.30/04_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41",
        "suitemasterfile/5.6.0.30/05_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41",
        "suitemasterfile/5.6.0.30/06_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41"
    ],
    "obtainedBondsRewardIds": []
}
```

3. 然后客户端则携带 `X-Session-Token` 先后访问了

```
GET /api/suitemasterfile/5.6.0.30/00_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41
GET /api/suitemasterfile/5.6.0.30/01_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41
GET /api/suitemasterfile/5.6.0.30/02_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41
GET /api/suitemasterfile/5.6.0.30/03_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41
GET /api/suitemasterfile/5.6.0.30/04_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41
GET /api/suitemasterfile/5.6.0.30/05_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41
GET /api/suitemasterfile/5.6.0.30/06_bb2194a8ea47722555dd056f8cc2a2d4563ea2041d4a3657b427285dca156e41
```

经测验，虽然携带了 `X-Session-Token`，但实际上需要的是包含完整 `CloudFront` 的 `Cookie` 

![image-20250915154830036](./Project-Sekai-逆向笔记.assets/image-20250915154830036.png)

*~~巨长的数据，编码后都有这么大，霓虹人太可怕了~~*

### [整活]强制 All Perfect

警告：此脚本存在封号风险，切勿使用

![image-20250927142928501](./Project-Sekai-逆向笔记.assets/image-20250927142928501.png)

```python
from mitmproxy import http
from time import time
from crypto import prsk_dec, prsk_enc
import json
from random import randint


def request(flow: http.HTTPFlow) -> None:
    # /api/user/<userId>/live/<userLiveId>
    if "/api/user/" in flow.request.pretty_url and "/live/" in flow.request.pretty_url:
        if flow.request.content is None:
            return
        body = prsk_dec(flow.request.content)
        if "score" in body:
            body["score"] = randint(90000, 120000)
            body["life"] = 1000
            perfectCount = body["perfectCount"]
            greatCount = body["greatCount"]
            goodCount = body["goodCount"]
            badCount = body["badCount"]
            missCount = body["missCount"]
            maxCombo = body["maxCombo"]
            total = perfectCount + greatCount + goodCount + badCount + missCount
            body["perfectCount"] = total
            body["greatCount"] = 0
            body["goodCount"] = 0
            body["badCount"] = 0
            body["missCount"] = 0
            body["maxCombo"] = total
            body["tapCount"] = total*2 + randint(0, 100)
            flow.request.content = prsk_enc(body)
            print(f"Modified live data: {body}")

```

### [整活] 自定义名片预览图

目标：上传自定义图片

观察发现用完名片后，会向 `https://production-game-api.sekai.colorfulpalette.org/api/user/{user_id}/custom-profile/1/custom-profile-card/{card_id}` 上传自定义名片数据，其中新增名片为 `POST`，修改名片为 `PUT`

Request 数据如下

```json
{
    "thumbnail": "此处为图片的预览图，格式为 JFIF base64，大小为 1190w * 528h",
    "customProfileCard": {
        "version": 3,
        "generals": [],
        "generalBackgrounds": [],
        "storyBackgrounds": [],
        "standMembers": [],
        "cardMembers": [],
        "honors": [],
        "bondsHonors": [],
        "texts": [
            {
                "objectData": {
                    "position": {
                        "x": -18.276344299316406,
                        "y": -13.701630592346191,
                        "z": 0
                    },
                    "scale": {
                        "x": 1,
                        "y": 1,
                        "z": 1
                    },
                    "rotation": {
                        "x": 0,
                        "y": 0,
                        "z": 0,
                        "w": 1
                    },
                    "layer": 0,
                    "lock": false,
                    "visible": true
                },
                "text": "hello",
                "fontId": 1,
                "type": 513,
                "colorId": 1,
                "size": 24,
                "outlineColorId": 1,
                "outlineSize": 0,
                "lineSpacing": 0
            }
        ],
        "collections": [],
        "others": [],
        "shapes": [],
        "stamps": []
    }
}
```

很简单，自己构造一个相同大小的 jpeg，b64 编码后上传即可

一个基本的 exp 如下

```python
# Patch custom-profile-card
from mitmproxy import http
from crypto import prsk_dec, prsk_enc # type: ignore

target_image_b64 = None
assert target_image_b64 is not None, "Please set target_image_b64 variable"

def request(flow: http.HTTPFlow) -> None:
    if flow.request.method in ["POST", "PUT"] and "custom-profile-card" in flow.request.pretty_url:
        if flow.request.content is None:
            return
        request_data = prsk_dec(flow.request.content)
        request_data['thumbnail'] = target_image_b64
        flow.request.content = prsk_enc(request_data)
```

成品图：

![image-20251018163712881](./Project-Sekai-逆向笔记.assets/image-20251018163712881.png)

继续可以搞一个 **大** 一点的图

先用 ffmpeg 切片并压缩

```bash
ffmpeg -i input.jpg -filter_complex \
"[0:v]crop=iw/3:ih/3:0:0[out1]; \
 [0:v]crop=iw/3:ih/3:iw/3:0[out2]; \
 [0:v]crop=iw/3:ih/3:2*iw/3:0[out3]; \
 [0:v]crop=iw/3:ih/3:0:ih/3[out4]; \
 [0:v]crop=iw/3:ih/3:iw/3:ih/3[out5]; \
 [0:v]crop=iw/3:ih/3:2*iw/3:ih/3[out6]; \
 [0:v]crop=iw/3:ih/3:0:2*ih/3[out7]; \
 [0:v]crop=iw/3:ih/3:iw/3:2*ih/3[out8]; \
 [0:v]crop=iw/3:ih/3:2*iw/3:2*ih/3[out9]" \
-map "[out1]" 1.jpg \
-map "[out2]" 2.jpg \
-map "[out3]" 3.jpg \
-map "[out4]" 4.jpg \
-map "[out5]" 5.jpg \
-map "[out6]" 6.jpg \
-map "[out7]" 7.jpg \
-map "[out8]" 8.jpg \
-map "[out9]" 9.jpg

```

exp

```python
from mitmproxy import http
from base64 import b64encode
from crypto import prsk_dec, prsk_enc # type: ignore

target_dir_path = '~/prsk/test/patchimages/'


# POST/PUT https://production-game-api.sekai.colorfulpalette.org/api/user/{user_id}/custom-profile/1/custom-profile-card/{card_id}
def request(flow: http.HTTPFlow) -> None:
    if flow.request.method in ["POST", "PUT"] and "custom-profile-card" in flow.request.pretty_url:
        if flow.request.content is None:
            return
        card_id = flow.request.pretty_url.split("/")[-1]
        request_data = prsk_dec(flow.request.content)
        target_image_path = target_dir_path + str(card_id) + '.jpg'
        with open(target_image_path, 'rb') as f:
            image = f.read()

        request_data['thumbnail'] = b64encode(image).decode('utf-8')
        flow.request.content = prsk_enc(request_data)
        # print(request_data) # check the modified request data but not actually send it
        print(f"Replaced thumbnail for card_id {card_id} from {target_image_path}")

```

成果图

![MuMu-20251018-165502-676](./Project-Sekai-逆向笔记.assets/MuMu-20251018-165502-676.png)

不过预览图应该是仅自己可见的，要想真改还得想办法注入名片

## PS 编写

尝试中，霓虹人定义的数据结构太恐怖了

## 第三方工具

### Sekai Viewer

众所周知的 [Sekai Viewer](https://sekai.best)

### SekaiTools

可用于 "统计原理、统计器、查看和编辑统计结果、系统语音、静态Spine场景、羁绊称号、互动语音" 等

参见 [Github](https://github.com/BLACKujira/SekaiTools) [Bilibili](https://www.bilibili.com/video/av381637796)

### 注入博客(以 hexo 为例)

参见 [在你的博客里放一只可爱的Spine Model吧~ | c10udlnk' Blog](https://c10udlnk.top/p/blogsFor-hexo-puttingLivelySpineModels/)

### 资源工具 sssekai

用于下载完整的未加密混淆的资源文件，参见 [GitHub](https://github.com/mos9527/sssekai) 

示例使用：

1. **构建 / 更新资源缓存索引**例：

```shell
sssekai abcache --db ./abcache.db --app-region jp --app-version 6.0.1 --app-appHash 0bd8114a-e782-681a-a54e-c5ec1ffb33b5
```

2. **下载资源包 / asset bundle**

```shell
sssekai abcache --db ./abcache.db --no-update --download-dir ./bundles/
```

3. **挂载资源 (Web)**

```shell
sssekai abserve --db ./abcache.db --host 127.0.0.1 --port 8831
```


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

参考 [Project SEKAI 逆向 - 笔记归档](https://mos9527.com/posts/pjsk/archive-20240105/)

研究版本：Project Sekai 国服 3.4.0

### 提取 `global-metadata.dat` (成功)

#### 游戏安装包(失败)

拿到公测开始公布的安装包，解压，找到`./assets/bin/Data/Managed/Metadata/global-metadata.dat`

发现 `AF 1B B1 FA`，但是在第九个字节开始，真的没加密吗？*~~我不信会这么仁慈~~*

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411153730114.png)

尝试把前八个字节删掉，并找到 `./lib/arm64-v8a/libil2cpp.so`，一并丢给 [Il2CppDumper](https://github.com/Perfare/Il2CppDumper/)

果不其然报错了，而且是运行一半之后才报错，如图

![](/image/articles/Project-Sekai-逆向笔记.assets/image-20250411154451449.png)

查看 `hexdump` 发现整个文件到处都是乱码，应该还是有加密

#### `/proc/[PID]/maps` 提取 `libcamera-metadata.dat`

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

![](/imag/articles/Project-Sekai-逆向笔记.assets/image-20250411192917347.png)

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

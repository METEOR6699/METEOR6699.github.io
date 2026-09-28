---
title: "libOTe：不经意传输扩展库详解"
featureimage: "images/covers/oblivious-transfer.png"
date: 2026-07-15
draft: false
tags: ["密码学", "不经意传输", "MPC"]
categories: ["隐私计算"]
description: "深入介绍 libOTe 库的实现原理与使用方式，涵盖不经意传输扩展、Silent OT 等核心概念。"
---

## 什么是不经意传输？

不经意传输（Oblivious Transfer, OT）是安全多方计算的基础协议之一。

**场景**：发送方有两条消息 m0 和 m1，接收方选择接收其中一条。协议保证：
- 接收方只能获得自己选择的消息
- 发送方不知道接收方选择了哪条

## OT 扩展（OT Extension）

基础 OT 协议通信开销大，OT 扩展技术可以用少量基础 OT 生成大量 OT 实例。

## libOTe 简介

libOTe 是俄亥俄州立大学（OSU）密码学组开发的高性能 C++20 OT 扩展库。

### 支持的协议

**半诚实模型**：
- 1-out-of-2 Silent OT
- 1-out-of-2 Correlated OT
- 1-out-of-N OT
- VOLE（向量不经意线性求值）

**恶意模型**：
- KOS15 协议
- Silent OT

### 特点

- 高性能：使用 SSE 指令和向量化优化
- 跨平台：支持 Windows、Linux、macOS
- 易用：提供简洁的 API 和前端示例

## 编译与使用

```bash
git clone https://github.com/osu-crypto/libOTe.git
cd libOTe
python3 build.py
```

### 基本示例

```cpp
#include "libOTe/TwoChooseOne/TcoOtExtReceiver.h"
#include "libOTe/TwoChooseOne/TcoOtExtSender.h"

// 创建 OT 扩展实例
OttExtReceiver* receiver = new IknpOtExtReceiver();
OttExtSender* sender = new IknpOtExtSender();

// 执行 OT 扩展
sender->genBaseOts(choice);
receiver->genBaseOts(choice);

std::vector<block> messages(128);
sender->send(messages, numOTs, prng, coproto::io);
receiver->receive(messages, choiceBits, numOTs, prng, coproto::io);
```

## 在隐私计算中的应用

OT 扩展是许多 MPC 协议的基础组件：

- **PSI（隐私集合交集）**
- **PIR（隐私信息检索）**
- **安全函数评估（SFE）**

---

**项目地址**：https://github.com/osu-crypto/libOTe

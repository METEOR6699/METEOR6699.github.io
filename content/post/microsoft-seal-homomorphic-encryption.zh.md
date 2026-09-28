---
title: "Microsoft SEAL 入门：同态加密实战"
featureimage: "images/covers/homomorphic-encryption.png"
date: 2026-07-16
draft: false
tags: ["密码学", "同态加密", "SEAL"]
categories: ["隐私计算"]
description: "Microsoft SEAL 同态加密库入门教程，从基本原理到代码实战，带你快速上手 BFV 和 CKKS 方案。"
---

## 什么是同态加密？

同态加密（Homomorphic Encryption）是一种特殊的加密方式，允许在密文上直接进行计算，解密后的结果与在明文上计算的结果相同。

简单来说：**数据加密后依然可以计算**。

## Microsoft SEAL 简介

Microsoft SEAL 是微软开发的开源同态加密库，支持以下三种方案：

- **BFV**：支持整数加法和乘法
- **CKKS**：支持近似浮点数运算
- **BGV**：支持整数运算，更适合批处理

## 快速上手

### 安装

```bash
git clone https://github.com/microsoft/SEAL.git
cd SEAL
cmake -S . -B build
cmake --build build
```

### 基本使用示例

```cpp
#include "seal/seal.h"
using namespace seal;

// 创建加密参数
EncryptionParameters params(scheme_type::ckks);
size_t poly_modulus_degree = 8192;
params.set_poly_modulus_degree(poly_modulus_degree);
coeff_modulus_id = CoeffModulus::Create(poly_modulus_degree, {60, 40, 40, 60});

// 创建 SEALContext
auto context = SEALContext::Create(params);

// 创建编码器、加密器、评估器
CKKSEncoder encoder(context);
Encryptor encryptor(context, key_generator.public_key());
Evaluator evaluator(context);
Decryptor decryptor(context, key_generator.secret_key());

// 编码、加密、计算、解密
std::vector<double> input = {0.5, 1.0, 1.5, 2.0};
Plaintext plain;
encoder.encode(input, scale, plain);

Ciphertext encrypted;
encryptor.encrypt(plain, encrypted);

// 在密文上进行加法
evaluator.add_inplace(encrypted, encrypted);
```

## 实际应用场景

1. **隐私保护的机器学习**：在加密数据上训练模型
2. **安全多方计算**：多方协作计算而不泄露各自数据
3. **云计算隐私**：将数据加密后上传云端计算

## 学习心得

同态加密的计算开销很大，但随着硬件加速和算法优化，正在逐步走向实用。SEAL 的文档和示例非常完善，是入门同态加密的好选择。

---

**项目地址**：https://github.com/microsoft/SEAL

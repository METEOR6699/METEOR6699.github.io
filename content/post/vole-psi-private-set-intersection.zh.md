---
title: "Vole-PSI：基于向量 OLE 的隐私集合交集"
featureimage: "images/covers/vole-psi.png"
date: 2026-07-14
draft: false
tags: ["密码学", "PSI", "隐私计算"]
categories: ["隐私计算"]
description: "深入解析 Vole-PSI 协议原理，介绍基于向量 OLE 的高效隐私集合交集实现方案。"
---

## 什么是隐私集合交集（PSI）？

隐私集合交集（Private Set Intersection）允许两方在不泄露各自集合内容的前提下，计算出两个集合的交集。

**应用场景**：
- 基因数据对比
- 联合风控
- 好友推荐
- 广告转化率统计

## Vole-PSI 的创新

Vole-PSI 基于向量不经意线性求值（Vector OLE）构建，相比传统 OT-based PSI 方案：

1. **通信效率更高**：向量 OLE 的通信开销与集合大小近似线性
2. **计算更快**：利用 OKVS（Oblivious Key-Value Store）减少计算量
3. **支持 Circuit PSI**：结果以秘密共享形式输出

## 技术原理

### 向量 OLE（Vector OLE）

向量 OLE 是 OT 的推广形式：
- 发送方输入向量 **a**，接收方输入标量 **b**
- 输出：发送方得到 **r**，接收方得到 **s** = **a** · **b** + **r**

### 协议流程

1. 双方通过 Vector OLE 生成关联的掩码值
2. 使用 OKVS 编码各自集合
3. 通过掩码值匹配交集元素

## 编译与使用

```bash
git clone https://github.com/Visa-Research/volepsi.git
cd volepsi
python3 build.py -DVOLE_PSI_ENABLE_BOOST=ON
```

### 前端使用

```bash
# 发送方
./frontend -role sender -set input_sender.txt -output result.txt

# 接收方
./frontend -role receiver -set input_receiver.txt -output result.txt
```

## 性能对比

| 方案 | 通信开销 | 计算时间（1M 元素） |
|------|----------|---------------------|
| ECDH-PSI | O(n) | ~10s |
| OT-based PSI | O(n) | ~5s |
| Vole-PSI | O(n) | ~2s |

## 学习心得

Vole-PSI 的代码结构清晰，依赖管理做得很好。通过学习这个项目，我对向量 OLE 和 OKVS 有了更深入的理解。

---

**项目地址**：https://github.com/Visa-Research/volepsi

---
title: "UPSI 论文复现实验手册：从 WSL2 环境到规模、带宽与结果统计"
slug: "upsi-reproduction-wsl2"
date: 2026-10-24
lastmod: 2026-10-24
draft: false
math: true
tags: ["PSI", "UPSI", "APSI", "论文复现", "WSL2", "Bazel"]
categories: ["隐私计算"]
description: "逐步复现 Efficient Updatable PSI from Asymmetric PSI and PSU：讲清 Bazel、YACL、RR22 与 APSI 的分工，解释预先建库与更新求交，附环境、编译、规模和网络测试命令。"
featureimage: "images/covers/privacy-computing.png"
---

这篇文章记录论文 **Efficient Updatable PSI from Asymmetric PSI and PSU** 的一次本地复现过程。目标是弄清楚实验如何运行、每个数字从哪里来，并在相近参数下比较规模和网络变化的影响。

本文采用 **Windows＋WSL2＋Ubuntu 22.04＋作者的 YACL 分支及 UPSI 源码**，以 **KRTW PSU＋BRPC** 为主线。安装、编译和执行的命令都写出执行位置，并解释关键参数。

先明确复现范围：论文在 CentOS 工作站的两个 Docker 容器中模拟两方；本文使用单个 WSL 环境，由示例程序在同一进程中并发模拟两方。两者硬件、调度和部分计时边界存在差异。**跑通相同协议和实验网格，不等于得到与论文完全一致的秒数，也不等于完成了安全性证明。** 本文不声称已复现论文的全部对比协议。

## 1. 先看懂实验究竟要测什么

### 1.1 初始求交和更新求交

双方持有集合 $X$、$Y$，希望得到交集 $I=X\cap Y$。普通 PSI 在执行后得到一次交集；UPSI 允许双方在集合变化后，利用已有状态更新交集。

用 $X^+$、$X^-$ 表示新增、删除的元素，更新后的集合为：

$$
X'=(X\setminus X^-)\cup X^+,
\qquad
Y'=(Y\setminus Y^-)\cup Y^+.
$$

UPSI 希望更新到 $I'=X'\cap Y'$，避免每次都对整个大集合重新做完整 PSI。本文使用的测试设定是：

$$
|X|=|Y|=N,\qquad
|X^+|=|Y^+|=|X^-|=|Y^-|=N^u.
$$

命令和目录名中的 `Nu` 就是论文中的 $N^u$，这里的上标 `u` 表示 update，并不是把 $N$ 再做一次数值乘方。

例如 `N20_Nu10` 表示：每方初始有 $2^{20}=1,048,576$ 个元素；每轮每方新增 $2^{10}=1,024$ 个、删除 $1,024$ 个。它不是“初始总共两方合计一百万”，也不是“总更新量只有一千”。

### 1.2 论文表一、表二各测什么

论文第 VI-A 节列出实验环境，第 VI-C 节及表 I、表 II 分别描述初始化与更新成本。[论文入口](https://ieeexplore.ieee.org/document/11361182)

| 项目 | 论文中的含义 | 本地记录时如何对应 |
|---|---|---|
| 表 I：初始化 | 建立可复用状态，并执行一次基础 PSI | 分别记录 `Setup time`、`Base PSI time`；本地汇总的初始化时间采用二者之和 |
| 表 II：更新 | 初始状态建立后，执行更新协议的时间与通信 | `UPSI time`、`UPSI Total Communication`；本教程用一轮更新作比较 |
| 表 III、表 IV 等 | 与 RR22、其他 UPSI 等方案的比较 | 需要单独准备对应协议，本文的脚本不会自动完成所有对比 |

论文说明初始化只需执行一次，后续更新性能不再计入这部分成本。**论文并没有写出日志字段名，也没有直接写出“Setup time＋Base PSI time”这条代码公式。** 这个加法是根据本版源码计时段建立的本地统计口径。

其中 `Setup` 在本版示例里先后建立两方 APSI 数据库，目的是为后续更新准备大集合查询状态；初始交集仍由 RR22 计算。它不是论文两容器并行运行的精确镜像。因此，把二者之和列入本地初始化表是可解释的，但不能据此断言它与论文的端到端初始化计时完全相同。

### 1.3 网络条件

论文和本次脚本使用以下名义条件：

| 脚本场景名 | 带宽 | RTT | 脚本设置的单向延迟 |
|---|---:|---:|---:|
| `LAN` | 1 Gbps＝1,000 Mbps＝$10^9$ bit/s | 0.2 ms | 0.1 ms |
| `WAN_200Mbps` | 200 Mbps | 80 ms | 40 ms |
| `WAN_50Mbps` | 50 Mbps | 80 ms | 40 ms |
| `WAN_5Mbps` | 5 Mbps | 80 ms | 40 ms |

这里的带宽单位是 **bit/s**。5 Mbps 不是 5 MB/s，换算成十进制字节吞吐上限约为 0.625 MB/s，实际有效吞吐还受协议和系统开销影响。`netem` 施加的是模拟条件，不能把配置值当成已经测得的应用吞吐量。

## 2. 相关知识点：APSI、RR22、PSU 和工具分别是什么

### 2.1 APSI 是什么，为什么 UPSI 需要它

**APSI 是 Asymmetric Private Set Intersection，即非对称隐私集合求交。** “非对称”指两方输入规模相差很大，例如一方有一百万条记录，另一方只查询一千条；它不是指 RSA 之类的公钥加密名称。

这个名字在本文中有三个层次：

1. 协议功能：小集合与大集合做隐私求交。
2. Microsoft 的 `APSI` C++ 库：提供该功能所需的数据库、查询、OPRF 和同态加密处理。
3. 作者 `aPSI.h` 中的 `class APSI`：对 Microsoft 库再做一层包装，供 UPSI 主程序调用。

UPSI 的原始大集合规模可能相同，例如双方都是一百万；但**每一轮的小规模新增集合与对方的大集合求交**，正是适合 APSI 的子问题。双方在整个协议中也不必永远保持同一个 sender/receiver 身份。

要区分两件事：**初始化时建立 APSI 数据库，更新时才调用 APSI 查询求交。** 建库是把自己的大集合预处理成以后可查询、可增删的状态，并不产生双方的初始交集。初始交集通过另外的 RR22 基础 PSI 获得。

本实验的依赖声明锁定 Microsoft **APSI v0.11.0**，并由作者 YACL 分支应用兼容补丁。不要拿最新 APSI 的接口或参数格式直接替换这版依赖。[对应版本说明](https://github.com/microsoft/APSI/tree/v0.11.0)

### 2.2 OPRF、同态加密和 SenderDB

- **OPRF**：不经意伪随机函数。查询方可获得自身输入的伪随机函数值，另一方处理请求时不直接看到这些明文输入。它不是给输入简单做一次普通哈希。
- **同态加密**：允许对密文做特定运算，解密结果对应明文计算。这里由 Microsoft SEAL 提供 BFV 方案相关能力。
- **BFV**：适用于模整数运算的同态加密方案。APSI 利用批处理、多项式计算等技术完成匹配。
- **SenderDB**：APSI 发送方预处理后的数据库。初始化时装入自己的完整集合，集合变动时在已有数据库上增删，供后续 APSI 查询使用；建库本身不计算双方交集。它是内存中的协议对象，不代表必须安装 MySQL 等数据库软件。
- **哈希分桶、bin**：把大量元素安排到多个桶，降低每个桶的匹配计算规模。
- **噪声预算**：同态密文支持计算的资源之一。运算会消耗预算；不是 Windows 内存，也不能靠任意缩小密码参数来节约资源。

### 2.3 RR22、KRTW 和 BRPC

| 名词 | 在本实验中的职责 | 不应混淆的地方 |
|---|---|---|
| PSI | 隐私集合求交功能 | 是问题类别，不是一种唯一实现 |
| UPSI | 保留状态、支持集合更新的 PSI | 不代表“再运行一次完整 PSI” |
| RR22 | 本版 `BasePsiSend/Recv` 使用的基础 PSI 路径 | 主要负责初始交集，不是 APSI 的另一个名字 |
| PSU | Private Set Union，隐私集合求并 | 更新时合并相关新增交集、删除信息；不把双方整个大集合直接公开 |
| KRTW | 一条具体的 PSU 协议实现路径 | `--backends=krtw` 选择的是 PSU 后端 |
| BRPC | RPC 通信框架，承载 YACL 链路消息 | `--link=brpc` 选择的是通信后端，不是 PSU 算法 |
| YACL | 密码、并行、通信等基础设施库 | 把各协议组件连接在一起 |
| OKVS / Baxos | 不经意键值存储及其实现，用于 RR22 相关编码 | 不是实验结果的 Excel 表，也不是 APSI 的 JSON 文件 |
| OT / VOLE | 不经意传输／向量不经意线性求值等密码学构件 | 由相关依赖和协议实现调用，不需逐个手工运行可执行文件 |
| IBLT | 可逆布隆查找表；仓库保留另一条 PSU 实现 | 本文固定 KRTW，避免把不同实现的结果混成一组 |

KRTW 名称来自 Kolesnikov、Rosulek、Trieu、Wang 等作者的姓氏缩写。本地适配器还处理双边获得并集结果的消息；应以本版实现及其安全模型理解这些输出。本文解释的是运行和测量路径，不用“交集检查通过”替代密码学安全审计。

### 2.4 编译、脚本、JSON 各是什么

**源码**是 `main.cc` 等 C++ 文件。编译器将它们与依赖构建、链接为 Linux 可执行程序 `upsi`。Bazel 管理这个过程；`BUILD.bazel` 描述该目标依赖哪些文件和库。

**Shell 脚本**负责反复启动程序、切换网络、整理日志。它在执行时解释运行，修改场景选择一般不需要重新编译 C++。

**JSON** 是文本数据格式。本实验中它保存 APSI 算法参数，程序启动后读取。它不是 Linux 命令，也不需要单独“运行 JSON”。

### 2.5 Bazel：负责把代码构建成程序

**Bazel 是构建和测试工具。** 在这个实验里，它读取项目的构建规则和依赖关系，安排下载依赖、调用 C++ 编译器和链接器，最后生成 `upsi`。它不执行 PSI 算法，也不在实验运行时决定交集。

几个容易混淆的名字可以这样对应：

| 名称 | 是什么 | 本实验中的作用 |
|---|---|---|
| Bazelisk | Bazel 的版本启动器 | 根据 `USE_BAZEL_VERSION=7.4.1` 找到并启动指定 Bazel |
| Bazel | 构建调度工具 | 根据规则组织编译、依赖构建和链接 |
| GCC / `g++` | C++ 编译器 | 把 C++ 源码编译为目标文件，再由相关工具链接 |
| `WORKSPACE` | 本版项目的工作区入口 | 声明工作区并引入外部依赖配置 |
| `BUILD.bazel` | 目录中的构建规则 | 指定 `upsi` 由哪些源码和库构成 |
| `.bazelrc` | Bazel 选项配置 | 保存优化选项、`--config=one` 等配置 |

例如命令 `bazel build //examples/upsi:upsi` 中，前面的 `bazel` 是工具，后面的 `//examples/upsi:upsi` 是要构建的**目标**。Bazel 会顺着该目标的依赖关系构建 YACL 模块及 APSI 等库，再生成 `bazel-bin/examples/upsi/upsi`。它不是把某一张实验表“编译进去”。

Bazel 会缓存已完成的构建工作：改 `main.cc` 后，只需重建受影响的部分；换网络时没有改 C++，可以用 `--skip-build` 直接运行原程序。构建中的 `--jobs=2` 也不是 APSI 运行时的线程数。[Bazel 官方介绍](https://bazel.build/about/intro)

### 2.6 YACL：程序实际使用的密码与通信基础库

**YACL 是 Yet Another Common crypto Library，一个 C++ 基础库。** 它提供密码学构件、数据类型、网络通信、I/O 等能力；本文使用作者维护的 YACL 分支，而非直接用上游最新版替换。

在本实验中，作者的 UPSI 代码位于 `yacl/examples/upsi`，会调用 YACL 的功能：

- 用 `uint128_t` 等类型表示实验元素，用随机数、哈希等工具构造或处理数据。
- 用 `yacl::link::Context` 表示通信端点，协议通过链路发送和接收消息，并读取通信量统计。
- 使用并行工具和底层密码学组件，支撑 RR22、PSU 及适配代码。
- 通过 `yacl_link_channel.h`，让 APSI 的消息进入本实验使用的 YACL 链路；`--link=brpc` 时由 BRPC 后端承载相应通信。

这些调用可从本版 `main.cc`、`upsi.cc`、`aPSI.h` 和 `BUILD.bazel` 核对。YACL 本身不是另一个需要在终端手工启动的 `yacl` 服务；相关模块经构建后成为 `upsi` 程序所依赖的代码。[YACL 上游项目说明](https://github.com/secretflow/yacl)

因此两者的分工是：**Bazel 负责把程序构建出来；YACL 提供程序运行时会调用的基础能力。** 作者把 RR22、APSI 和 KRTW PSU 组合成 UPSI，Bazel 再根据构建规则把这些代码及其依赖构建为可执行程序。

## 3. 各组件怎样一起工作

### 3.1 从一条命令到一组结果

{{< mermaid >}}
flowchart TD
    A[run_benchmarks.sh 解析实验参数] --> B[隔离网络内设置 tc/netem]
    B --> D[启动已编译的 upsi]
    D --> E[Setup: 用初始 X 和 Y 建立 APSI 数据库]
    J[APSI JSON 参数] --> E
    E -->|源码接下来执行| F[Base PSI: RR22 直接对 X 和 Y 求交]
    F --> G[保存并校验初始交集 I0]
    E -. 保留供后续查询使用 .-> S[(两方 APSI 数据库状态)]
    G --> H[新一轮: 按新增和删除批次修改数据库]
    S --> H
    H --> Q[APSI: 新增小集合与对方更新后的大集合求交]
    Q --> P[KRTW PSU 合并更新信息并更新旧交集]
    P --> I[校验更新交集并输出日志]
    I --> K[逐次记录与汇总文件]
{{< /mermaid >}}

构建是图外的准备工作：Bazel 先把 C++ 源码和依赖生成 `upsi`。使用 `--skip-build` 后，批量脚本只启动现有程序，不再构建。

图中“建库 → RR22”的实线表示本版源码的执行顺序，**不表示 RR22 要以 APSI 数据库为输入**。RR22 接收的是原始集合 `X`、`Y`；虚线表示 APSI 数据库状态要保留，供后续更新使用。

### 3.2 程序内部的阶段

本版入口是 `main.cc` 的 `RunUPSIv1`。按实际调用顺序可读成：

```text
构造测试元素、预期交集、Baxos 参数
    ↓
Setup 计时开始
    建立 instanceX、instanceY 两个 APSI 对象
    把初始集合插入两方 SenderDB，预备后续更新查询
    此时还没有通过 APSI 计算双方交集
Setup 计时结束
    ↓
建立 YACL 链路，将 APSI Channel 适配到该链路
    ↓
Base PSI 计时开始
    并发执行 RR22 发送方与接收方，输入为 X 和 Y
    得到并保存初始交集 I0，不读取 APSI 数据库来求交
Base PSI 计时结束
    检查双方初始交集，打印基础通信量
    ↓
对每一轮更新：
    构造新增/删除测试数据
    UPSI 本轮计时开始
        在已有 APSI 数据库中删除旧元素、插入新元素
        执行双向 APSI 查询，处理新增小集合的匹配
        执行新增相关的 KRTW PSU，合并新增交集
        从旧交集中提取删除相关信息
        执行删除相关的 KRTW PSU，合并需移除的信息
        更新双方保存的交集
    UPSI 本轮计时结束
    检查双方结果与预期交集
    打印本轮时间、通信量
    ↓
打印所有更新轮次的总时间、总通信量
```

这里的 `Setup` 不包括前面所有数据生成、Baxos 初始化，也不包括后面所有链路建立工作；“程序总用时”因此不能只靠三个字段严格还原。

本版测试数据由程序生成，不用先准备 CSV 数据集。新增、删除批次也是为测试构造的；它们不能代表所有真实业务的数据分布。

### 3.3 为什么先建 APSI 数据库，初始交集却用 RR22

因为初始化要为以后准备**两类状态**，职责不同：

| 初始化工作 | 准备的状态 | 后续用途 |
|---|---|---|
| 将自己的初始集合插入 APSI 的 SenderDB | `DB_X`、`DB_Y`，即两方大集合的可查询表示 | 集合变化时增删，再回答小规模新增集合的 APSI 查询 |
| 对初始集合执行 RR22 基础 PSI | 双方保存的初始交集 `I0` | 更新时在旧交集上删除失效匹配、加入新匹配 |

**建立数据库不等于执行一次 APSI 求交。** 可以把它理解为先把一本大字典整理好，等后续拿着少量新词来查；整理字典并不等于已经比较了双方的全部词汇。

本版源码先建库，再执行 RR22，是实现选择的先后顺序。决定谁计算初始交集的，是调用了哪个求交函数，而不是哪种数据库先建立。

下面摘取 `RunUPSIv1` 的关键调用，省略链路建立、计时和校验细节。它用于理解调用关系，不是一段可独立编译的完整程序：

```cpp
// 1. 为以后准备 APSI 大集合状态；这几行不产生双方交集。
APSI instanceX(ResolveParamsPath("1M-1024-com.json"));
APSI instanceY(ResolveParamsPath("1M-1024-com.json"));
instanceX.insertItems(X);
instanceY.insertItems(Y);

// 2. 初始交集由基础 PSI 完成，两方需要并发执行。
// BasePsiSend / BasePsiRecv 内部调用 RR22PsiSend / RR22PsiRecv。
auto rr22_sender = std::async(std::launch::async, [&] {
  return BasePsiSend(lctxs[0], X, baxos);
});
auto rr22_receiver = std::async(std::launch::async, [&] {
  return BasePsiRecv(lctxs[1], Y, baxos);
});
auto initial_sender = rr22_sender.get();
auto initial_receiver = rr22_receiver.get();

// 3. 更新轮先维护已有数据库，不重新从全部 N 个元素建库。
instanceX.deleteItems(batch.sub_items);
instanceX.insertItems(batch.add_items);
instanceY.deleteItems(batch.sub_items);
instanceY.insertItems(batch.add_items);

// 随后进入 UPsiSendV1 / UPsiRecvV1：APsiRun 处理新增项查询，
// PsuSend / PsuRecv 合并更新信息，结合旧交集生成新交集。
```

例如 N 约一百万、Nu 为 1024：初始化先把各自一百万元素预处理进数据库；RR22 再获得初始交集。下一轮每方删除、插入各 1024 个元素时，程序修改已有数据库，然后让新增小集合与对方更新后的大集合执行 APSI 查询，配合 PSU 更新旧交集。

如果等每轮更新时才从零建立这两个数据库，就会反复处理完整大集合，失去复用预处理状态的意义。因此，Setup 阶段建库的目的很明确：**提前准备未来查询用的状态，初始全量求交仍由传统 PSI 的 RR22 完成。**

还有一个区别：数据库删除是本地状态维护；更新协议中的删除处理还要利用旧交集和 PSU。不能把“APSI 数据库里删除了元素”理解为双方交集已经自动完成同步更新。

### 3.4 为什么不能把每个组件当独立软件依次启动

执行 `upsi` 后，RR22、APSI、PSU 是由 C++ 函数调用连接的。SEAL、libOTe 等依赖经 Bazel 构建并链接，某些对象是程序内部的状态。

`--link=brpc` 时，相关 RR22、APSI 及 KRTW 通信通过 YACL 链路在本机回环路径上交换，`tc/netem` 对该路径施加带宽和延迟。`--link=memory` 则主要用于内存通信功能检查，不能拿它的结果代替 WAN 测量。

脚本总会打印一段 IBLT 网络限制提示。即使选择 KRTW，也可能看到这条通用说明；它不表示程序偷偷切换到了 IBLT，更不是单独的失败判据。

## 4. 环境准备：分清 PowerShell 和 Ubuntu

### 4.1 哪个窗口执行什么

| 提示符示例 | 环境 | 在这里执行 |
|---|---|---|
| `PS C:\Users\用户名>` | Windows PowerShell | 安装／启动 WSL、编辑 `.wslconfig` |
| `用户名@电脑名:~$` | Ubuntu Bash | `apt`、`git`、`bazel`、实验脚本 |

代码块不带提示符，复制时只复制命令。Ubuntu 中 `~` 是 `/home/你的用户名`；`/mnt/f` 对应 Windows 的 F 盘。`cd` 只是切换当前目录。

### 4.2 安装 WSL2 和 Ubuntu 22.04

**仅首次安装时，在管理员 PowerShell 执行：**

```powershell
wsl --install --no-distribution
```

如果提示重启，先重启 Windows。随后在 PowerShell 查看发行版并安装：

```powershell
wsl --list --online
wsl --install -d Ubuntu-22.04
```

首次进入 Ubuntu 时创建 Linux 用户名和密码。输入密码时不显示字符是正常现象。检查版本：

```powershell
wsl --list --verbose
```

对应发行版应显示 `VERSION 2`。已安装并正常运行的环境不需要重复安装。[WSL 官方命令说明](https://learn.microsoft.com/en-us/windows/wsl/basic-commands)

### 4.3 希望大部分空间占用放到 F 盘怎么办

`apt install` 默认安装到 Linux 的 `/usr` 等目录，不是给每个包选择 Windows 的安装盘。**决定这些包主要占哪个盘的是发行版虚拟磁盘的位置。** 仅把实验输出放到 `/mnt/f`，并不会自动把原来位于 C 盘的 Ubuntu 软件包搬走。

如果从零建立新环境，且 `wsl --help` 中包含 `--location`，可以在 PowerShell 使用下面这条安装命令，替代上一节的发行版安装命令：

```powershell
wsl --install -d Ubuntu-22.04 --location F:\WSL\Ubuntu-22.04
```

这不是给已安装发行版执行的迁移命令。已有环境先保持原状即可；本文统一把实验结果写入 `F:\PSI-results\UPSI`。源码和构建放在 Linux 家目录，避免直接在 `/mnt/f` 的 Windows 文件系统上做大量小文件编译。

### 4.4 限制 WSL 资源

本次笔记本有 16 GB 物理内存，实验环境曾给 WSL 约 8 GiB 内存、4 个逻辑处理器和 4 GiB swap。配置上限不代表实验必然能在这些资源中成功。

在 **PowerShell** 打开配置文件：

```powershell
notepad "$env:USERPROFILE\.wslconfig"
```

如果已有配置，只调整相应项，不要覆盖其他已有设置。示例：

```ini
[wsl2]
memory=8GB
processors=4
swap=4GB
```

保存后，确保没有未保存的 Linux 工作，再在 PowerShell 执行：

```powershell
wsl --shutdown
wsl -d Ubuntu-22.04
```

`wsl --shutdown` 会停止正在运行的 WSL 工作，不能在实验进行中随意执行。进入 Ubuntu 检查：

```bash
cd ~
free -h
nproc
```

`free -h` 看内存和交换空间；`nproc` 看当前可用处理器数。它们是两条命令，不要拼成 `free -h nproc`。

## 5. 安装构建工具和固定版本源码

### 5.1 Ubuntu 基础工具

下面开始，除非注明 PowerShell，全部在 **Ubuntu** 执行：

```bash
sudo apt update
sudo apt install -y \
  build-essential cmake ninja-build git curl ca-certificates \
  autoconf automake libtool m4 make patch perl pkg-config \
  python3 unzip zip iproute2 util-linux binutils
```

这些是编译器、构建工具、下载工具及网络命令。`iproute2` 提供 `ip`、`tc`，`util-linux` 提供脚本使用的 `unshare`。

APSI、SEAL、Kuku、libOTe、volePSI 等 C++ 依赖由作者的 Bazel 配置管理。**不需要再逐个 clone、安装到 `/usr/local`。** 手工混入其他版本反而会增加接口和链接冲突。

### 5.2 安装 Bazel 7.4.1

本次使用 x86-64 Ubuntu 和 Bazel 7.4.1。这里通过固定版本 Bazelisk 启动它：

```bash
mkdir -p "$HOME/.local/bin"

curl -fL \
  https://github.com/bazelbuild/bazelisk/releases/download/v1.22.1/bazelisk-linux-amd64 \
  -o "$HOME/.local/bin/bazel"

chmod +x "$HOME/.local/bin/bazel"
export PATH="$HOME/.local/bin:$PATH"
export USE_BAZEL_VERSION=7.4.1

bazel --version
g++ --version | head -n 1
cmake --version | head -n 1
```

`mkdir -p` 创建目录；`curl -o` 下载到指定文件；`chmod +x` 增加执行权限。`PATH` 决定输入 `bazel` 时从哪里查找程序；`USE_BAZEL_VERSION` 告诉 Bazelisk 使用哪个 Bazel 版本。

这些 `export` 默认只影响当前 Shell 及其子进程。重新开终端后要重新执行，或者自行写入 `~/.bashrc`。它们不是修改 UPSI 算法。

### 5.3 代理是下载通道，不是论文网络参数

能直接访问 GitHub 时，跳过本节。如果 Windows 的代理软件监听 HTTP 端口 `7892`，可以先在 **Ubuntu** 测试：

```bash
curl -I -x http://127.0.0.1:7892 --max-time 15 https://github.com
```

`127.0.0.1` 这种连接方式要求相应 WSL 网络模式支持访问 Windows 的 localhost，并且代理确实在该地址、端口监听。NAT 模式通常需要使用 Windows 主机地址，并由代理允许相应连接。不要为了下载代码盲目反复改网络配置。[WSL 网络说明](https://learn.microsoft.com/en-us/windows/wsl/networking)

确认上面的测试成功后，可以在当前 Ubuntu Shell 设置：

```bash
export http_proxy="http://127.0.0.1:7892"
export https_proxy="http://127.0.0.1:7892"
export no_proxy="localhost,127.0.0.1,::1"
```

HTTPS 网站通过 HTTP 代理建立隧道，所以这里 `https_proxy` 的值仍可为 `http://...`。换代理软件后只在地址、端口或协议变化时调整对应值。

下载完成后若无需代理，可在当前 Shell 取消：

```bash
unset http_proxy https_proxy
```

代理用于访问 GitHub 和下载依赖，**不会把 `WAN_5Mbps` 变成真实跨国网络实验**。实验网络由后面的隔离命名空间与 `tc` 设置。

### 5.4 拉取并固定作者源码

```bash
mkdir -p ~/psi-paper
cd ~/psi-paper

git clone https://github.com/ShallMate/yacl.git
git -C yacl checkout 46d6137b297d9ac09fcb8401ddd89e705bcba175

git clone https://github.com/ShallMate/upsi.git
git -C upsi checkout e3dd3731f29c791d686c464f6fe48979db025130

test ! -e yacl/examples/upsi
mkdir -p yacl/examples
mv upsi yacl/examples/upsi

cd yacl
test -f WORKSPACE
test -f examples/upsi/BUILD.bazel
```

这些命令用于新目录。若已有同名仓库，不要重复克隆或覆盖。`test ! -e` 用来检查目标不存在；若该检查失败，应先查看已有目录再处理，不要直接继续执行 `mv`。

Git 的长字符串是 **commit 提交号**。固定它能避免下次 clone 时默认分支变动而产生不同结果。切换具体提交时出现 `detached HEAD` 是正常状态，可以编译和运行。

如果直接 clone 遇到 HTTP/2 或 TLS 错误，且已确认上述代理可用，可以将对应的 clone 命令改为：

```bash
git -c http.proxy=http://127.0.0.1:7892 \
    -c http.version=HTTP/1.1 \
    clone https://github.com/ShallMate/upsi.git
```

`git -c` 的配置只对这一条 Git 命令生效，不会永久修改 Git 全局设置。这里仅以 UPSI 仓库举例，另一个仓库遇到同类问题时同理。

### 5.5 现在的目录应该是什么样

```text
~/psi-paper/
└── yacl/                         # Bazel 工作区
    ├── WORKSPACE
    ├── .bazelrc
    ├── bazel/                    # 第三方依赖与补丁
    ├── yacl/                     # 密码和通信基础设施
    └── examples/upsi/            # 嵌入的独立 UPSI Git 仓库
        ├── BUILD.bazel
        ├── main.cc               # 规模、JSON选择、计时、测试入口
        ├── upsi.cc               # 基础PSI及更新协议调用
        ├── aPSI.h / aPSI.cc       # APSI包装
        ├── yacl_link_channel.h   # APSI消息到YACL链路的适配
        ├── rr22/
        ├── psu/
        ├── parameters/           # JSON参数
        ├── run_benchmarks.sh
        └── network_setup.sh
```

两个仓库是嵌套关系，查看修改时要注意位置：

```bash
git -C ~/psi-paper/yacl rev-parse HEAD
git -C ~/psi-paper/yacl/examples/upsi rev-parse HEAD
git -C ~/psi-paper/yacl/examples/upsi status --short
```

## 6. 第一次编译：先生成能运行的程序

在 **YACL 根目录**执行：

```bash
cd ~/psi-paper/yacl
export PATH="$HOME/.local/bin:$PATH"
export USE_BAZEL_VERSION=7.4.1

bazel build \
  --config=one \
  --jobs=2 \
  --local_resources=memory=4096 \
  //examples/upsi:upsi
```

| 部分 | 含义 |
|---|---|
| `bazel build` | 构建目标及其依赖 |
| `--config=one` | 使用作者 `.bazelrc` 中面向单个目标的配置，清除默认全目标列表 |
| `--jobs=2` | 最多安排两个构建任务；不是运行时 APSI 线程数 |
| `--local_resources=memory=4096` | 告诉 Bazel 本地调度可用内存资源；不是对所有进程施加 4 GiB 硬限制 |
| `//examples/upsi:upsi` | 工作区中 `examples/upsi/BUILD.bazel` 定义的 `upsi` 目标 |

反斜杠 `\` 表示 Bash 命令换行，后面不能再放空格。编译成功后应看到 `Build completed successfully`，程序位于：

```bash
ls -lh bazel-bin/examples/upsi/upsi
bazel-bin/examples/upsi/upsi --help
```

`bazel-bin` 等目录可能是 Bazel 建立的链接。首次构建要下载和编译很多依赖，后续增量构建通常快得多。

原始提交默认是 `N=2^20`、`Nu=2^9`、`1M-512-com.json`。如果是已有实验目录，先检查源码，不要仅凭过去的命令猜当前程序规模：

```bash
sed -n '/void RunUPSIv1/,/auto lctxs/p' examples/upsi/main.cc
```

## 7. 先完成一组小规模流程，再批量运行网络

### 7.1 一次 LAN 验证

```bash
cd ~/psi-paper/yacl/examples/upsi

out_dir="/mnt/f/PSI-results/UPSI/N20_Nu9_LAN_$(date +%Y%m%d_%H%M%S)"

./run_benchmarks.sh \
  --backends=krtw \
  --scenarios=LAN \
  --repeats=1 \
  --update-count=1 \
  --link=brpc \
  --skip-build \
  --output-dir="$out_dir"

cat "$out_dir/summary.md"
```

前提是上一节构建的确实为原始默认规模。`out_dir` 只是输出目录变量，目录名写 `N20_Nu9` **不会修改程序实际规模**。`$(date ...)` 在运行命令时生成时间戳，避免覆盖不同批次。

`--skip-build` 很重要：这里已经在工作区根目录显式完成构建，运行脚本时复用该程序。

### 7.2 同一规模，四种网络各测三次

```bash
cd ~/psi-paper/yacl/examples/upsi

out_dir="/mnt/f/PSI-results/UPSI/N20_Nu9_$(date +%Y%m%d_%H%M%S)"

./run_benchmarks.sh \
  --backends=krtw \
  --scenarios=LAN,WAN_200Mbps,WAN_50Mbps,WAN_5Mbps \
  --repeats=3 \
  --update-count=1 \
  --link=brpc \
  --skip-build \
  --output-dir="$out_dir"
```

| 参数 | 含义 |
|---|---|
| `--backends=krtw` | 固定 PSU 实现，避免脚本默认同时运行其他后端 |
| `--scenarios=...` | 选择脚本预定义的网络条件 |
| `--repeats=3` | 每个后端与网络组合，独立启动程序三次 |
| `--update-count=1` | 每次程序运行，初始化后执行一轮更新 |
| `--link=brpc` | 用真实链路后端，供网络模拟影响消息传输 |
| `--skip-build` | 只运行已有程序 |
| `--output-dir=...` | 这整批结果保存到哪里 |

总共是 **1 个 PSU 后端×4 种网络×3 次独立运行＝12 次程序启动**。每次都从头初始化，再完成一轮更新。

### 7.3 repeats 与 update-count 为什么不矛盾

```text
repeats=3，update-count=1：
    第1次：初始化 → 更新1轮 → 退出
    第2次：初始化 → 更新1轮 → 退出
    第3次：初始化 → 更新1轮 → 退出

repeats=1，update-count=3：
    第1次：初始化 → 更新第1轮 → 更新第2轮 → 更新第3轮 → 退出
```

第一种用来观察独立重复的波动。第二种观察状态连续更新；后三轮共享先前状态。`UPSI time` 在多轮情况下是更新时间之和，不能直接拿“三轮总时间”与论文“一轮更新”表格比较。

### 7.4 只更换网络，不需要重新编译

例如只测 50 Mbps：

```bash
./run_benchmarks.sh \
  --backends=krtw \
  --scenarios=WAN_50Mbps \
  --repeats=3 \
  --update-count=1 \
  --link=brpc \
  --skip-build \
  --output-dir="/mnt/f/PSI-results/UPSI/N20_Nu9_WAN50_$(date +%Y%m%d_%H%M%S)"
```

`WAN_50Mbps` 是 Shell 脚本中关联数组的键，选出 `RTT=80`、`rate=50`，再传给网络脚本。它不是 C++ 函数名，更不是下载另一个程序。

`network_setup.sh` 最终执行类似下面的操作：

```bash
# 原理示意，不要在普通宿主命名空间中另外执行这段。
tc qdisc replace dev lo root netem delay 40.000ms rate 50mbit
```

批量脚本已经负责在其命名空间内应用和清理网络设置。不要再在别的终端重复手工设置 `tc`，否则可能设置在另一个命名空间，或影响其他本地流量。

## 8. 改规模：到底需要修改哪几处

### 8.1 需要修改的代码

在 `main.cc` 中找到 **`RunUPSIv1`**，不是其他演示函数。例如把默认规模改为 `N=2^20、Nu=2^10`：

```cpp
const uint64_t num = 1 << 20;
const uint64_t addnum = 1 << 10;
const uint64_t subnum = 1 << 10;
```

以及同一函数中的两行：

```cpp
APSI instanceX(ResolveParamsPath("1M-1024-com.json"));
APSI instanceY(ResolveParamsPath("1M-1024-com.json"));
```

`1 << 20` 是整数左移，对这些正整数等价于 $2^{20}$。`num` 控制初始集合规模，`addnum`、`subnum` 分别控制每方每轮的新增和删除数量。

两处 JSON 分别用于两个 APSI 实例。这组对称规模实验中应一起修改，不能只改 X 不改 Y。代码里另外的演示函数可能也出现 JSON 字符串，不要全文件无差别替换。

### 8.2 可检查的自动修改命令

不熟悉编辑器时，可以在 **YACL 根目录**执行以下 Python 辅助命令。它只处理 `RunUPSIv1` 这段，并检查修改次数；辅助代码用于切换实验配置，不改变作者协议流程。

```bash
cd ~/psi-paper/yacl

N_EXP=20 NU_EXP=10 PARAM_FILE=1M-1024-com.json python3 - <<'PY'
import os
import re
from pathlib import Path

n_exp = int(os.environ["N_EXP"])
nu_exp = int(os.environ["NU_EXP"])
param = os.environ["PARAM_FILE"]
assert 1 <= nu_exp <= n_exp <= 30
assert re.fullmatch(r"[A-Za-z0-9._-]+\.json", param)

source = Path("examples/upsi/main.cc")
assert (source.parent / "parameters" / param).is_file()
text = source.read_text()
start = text.index("void RunUPSIv1(")
end = text.index("\nint main(", start)
body = text[start:end]

for name, exponent in [("num", n_exp), ("addnum", nu_exp), ("subnum", nu_exp)]:
    body, count = re.subn(
        rf"const uint64_t {name} = 1 << \d+;",
        f"const uint64_t {name} = 1 << {exponent};",
        body,
    )
    assert count == 1, (name, count)

body, count = re.subn(
    r'(APSI instance[XY]\(ResolveParamsPath\(")[^"]+("\)\);)',
    lambda match: match[1] + param + match[2],
    body,
)
assert count == 2, count

# 留下第一次修改前的源码副本，不覆盖已有备份。
backup = source.with_name("main.cc.before-scale-change")
if not backup.exists():
    backup.write_text(text)
source.write_text(text[:start] + body + text[end:])
print(f"Configured N=2^{n_exp}, Nu=2^{nu_exp}, JSON={param}")
PY

git -C examples/upsi diff -- main.cc
```

要改的只有开头三个值：`N_EXP` 是 N 的指数，`NU_EXP` 是 Nu 的指数，`PARAM_FILE` 是选择的 JSON 文件名。`re.subn` 返回实际替换次数；`assert` 用来防止源码结构变化后误改其他地方。

注意：此命令**不修改 APSI 全局线程数**。要保持前后结果可比，应单独确认 `ThreadPoolMgr::SetThreadCount(...)` 沿用同一策略。

### 8.3 修改后必须重新编译

```bash
cd ~/psi-paper/yacl

bazel build \
  --config=one \
  --jobs=2 \
  --local_resources=memory=4096 \
  //examples/upsi:upsi
```

规模常量和 JSON **文件名**写在 C++ 源码中，所以修改后要重新编译。只改源码、不构建，再运行 `--skip-build`，执行的仍可能是旧规模。

JSON **内容**是在启动时读取的：如果文件名不变，只改其内容，通常不需重新编译。但这样会改变同一程序的实验条件；必须另存参数副本、记录哈希，不能悄悄覆盖正式实验的原 JSON。

## 9. JSON 文件怎么选、怎么看

### 9.1 本次网格使用的参数文件

以下是本次本地复现的配置记录，不宣称是论文每一行的唯一参数映射：

| N | Nu | APSI 参数文件 | 当前记录 |
|---:|---:|---|---|
| $2^{18}$ | $2^9$ | `256K-512.json` | 已测 |
| $2^{18}$ | $2^{10}$ | `256K-1024.json` | 已测 |
| $2^{18}$ | $2^{11}$ | `256K-2048-com.json` | 已测 |
| $2^{18}$ | $2^{12}$ | `256K-4096-com.json` | 已测，含失败记录 |
| $2^{20}$ | $2^9$ | `1M-512-com.json` | 已测 |
| $2^{20}$ | $2^{10}$ | `1M-1024-com.json` | 已测 |
| $2^{20}$ | $2^{11}$ | `1M-2048-com.json` | 已测 |
| $2^{20}$ | $2^{12}$ | `1M-4096-com.json` | 已测，含失败记录 |
| $2^{20}$ | $2^{13}$ | `1M-11041.json` | 已测 |
| $2^{22}$ | $2^{10}$ | `16M-1024.json` | 已测 |
| $2^{22}$ | $2^{11}$ | `16M-2048.json` | 已测，含失败记录 |
| $2^{22}$ | $2^{12}$ | `16M-4096.json` | 当前汇总标为未测，原失败数据已清理 |
| $2^{22}$ | $2^{13}$ | `16M-11041.json` | 未测，计划参数 |

同一行四种网络使用同一份 JSON；网络条件由脚本改变。文件名不是程序的 N、Nu 输入接口：`1M` 通常表示面向百万量级发送方的参数族；`512` 是相关查询规模的命名提示，不等于运行时硬编码 Nu。

`-com` 可理解为参数族中偏通信优化的命名提示，最终仍要看文件实际字段，不能把后缀当成可执行开关。`11041` 也不是把实际 Nu 设置成 11041；本次仍可令 Nu 为 8192。

对于 N=$2^{22}$，本次使用作者现有的 `16M-*` 参数族。**文件名 16M 不会把程序 N 改成 $2^{24}$；同时，参数容量覆盖实际规模不等于已确认与论文选用的 JSON 完全相同。** 这个差异必须保留在实验说明中。

### 9.2 以 1M-512-com.json 为例

在 Ubuntu 查看并校验 JSON 语法：

```bash
cd ~/psi-paper/yacl/examples/upsi
python3 -m json.tool parameters/1M-512-com.json
```

本版文件内容为：

```json
{
  "table_params": {
    "hash_func_count": 3,
    "table_size": 1364,
    "max_items_per_bin": 228
  },
  "item_params": {
    "felts_per_item": 6
  },
  "query_params": {
    "ps_low_degree": 4,
    "query_powers": [1, 2, 3, 4, 5, 10, 15, 35, 55, 75, 95, 115, 125, 130, 140]
  },
  "seal_params": {
    "plain_modulus": 65537,
    "poly_modulus_degree": 4096,
    "coeff_modulus_bits": [48, 34, 27]
  }
}
```

| 字段 | 直观含义 | 为什么影响性能 |
|---|---|---|
| `hash_func_count` | 哈希分桶使用的哈希函数数量 | 影响分桶布局和相关处理 |
| `table_size` | APSI 哈希表槽位数 | 影响查询批量、密文数量与计算工作 |
| `max_items_per_bin` | 每个桶支持的元素规模参数 | 与匹配多项式复杂度、分包等有关 |
| `felts_per_item` | 一个元素编码成多少个有限域元素 | 影响表示长度与匹配工作量 |
| `ps_low_degree` | Paterson–Stockmeyer 多项式求值的低阶划分参数 | 改变乘法安排和计算代价 |
| `query_powers` | 查询方预先提供的幂次集合 | 决定通信与服务端计算之间的取舍 |
| `plain_modulus` | BFV 明文模数 | 决定模整数计算空间，也影响噪声消耗 |
| `poly_modulus_degree` | 同态加密多项式环维度 | 影响批处理容量、计算、内存和安全约束 |
| `coeff_modulus_bits` | 密文系数模数链各素数的位数 | 影响噪声预算、密文大小及安全约束 |

JSON 里的 `table_size=1364` 不是 N，也不是 Nu。文件中没有直接写 `N=1048576`，不代表参数与集合规模无关。

这些字段共同组成密码学参数，不能只为变快而任意降低。若 Nu 增大时换了 JSON，即同时改变了更新工作量和算法参数；时间不必单调增长。要验证规模影响，应固定同一 JSON 做对照，并说明该参数对查询规模的适用边界。

## 10. 每一类输出文件怎么读

一批运行完后大致得到：

```text
N20_Nu10_时间戳/
├── execution.log
├── details.tsv
├── summary.tsv
├── summary.md
└── runs/
    ├── krtw_LAN_run1.log
    ├── krtw_LAN_run2.log
    ├── krtw_LAN_run3.log
    ├── krtw_WAN_200Mbps_run1.log
    └── ……其余逐次日志
```

| 文件 | 记录内容 | 什么时候看 |
|---|---|---|
| `execution.log` | 批量脚本的开始／结束、网络设置、退出状态；由脚本构建时也记录构建输出 | 看整批调度和网络设置是否成功 |
| `runs/*.log` | 某一次 C++ 程序的完整标准输出和错误输出 | 核对真实规模、计时、交集检查、异常原因 |
| `details.tsv` | 每次运行一行的指标、网络、状态、退出码、日志路径 | 做逐次统计和追溯 |
| `summary.tsv` | 每个后端与网络组合一行的均值、Std 等 | 导入表格、批量分析 |
| `summary.md` | 人可以直接读的 Markdown 汇总与逐次表 | 终端快速查看 |

`.tsv` 使用制表符分列；Excel/WPS 可以按制表符导入。`.md` 是 Markdown 文本，`cat` 看到竖线分隔的原始文本属于正常现象。

跑四种网络仍然只有一份 `details.tsv`、一份 `summary.tsv`、一份 `summary.md`，是因为它们按整批汇总；网络通过行中的 `scenario` 区分。不是只保存了三个实验。

查看命令：

```bash
cat "$out_dir/summary.md"
column -s $'\t' -t "$out_dir/details.tsv" | less -S
less "$out_dir/runs/krtw_LAN_run1.log"
```

`less` 按 `q` 退出。`$'\t'` 是 Bash 中表示制表符的写法；双引号保护含空格的目录变量。

## 11. 常见故障各意味着什么

| 现象 | 能确认什么 | 不能直接断言什么 |
|---|---|---|
| `failed to remove item` | APSI 数据库删除路径抛出异常，需结合完整日志定位 | 不是看见这句就能认定是带宽、线程或内存造成 |
| Zstandard compression failed | 数据压缩路径失败 | 不能只凭名称认定网络坏了 |
| `Cannot allocate memory` / `std::bad_alloc` | 某次内存或地址空间分配受限 | 不一定只由物理 RAM 耗尽，也可能与上限等有关 |
| 退出码 `134` | 通常为进程收到 SIGABRT 后异常中止 | 退出码本身不能说明具体哪段代码错误 |
| 退出码 `137` | 通常与 SIGKILL 对应 | 需要日志判断是否 OOM、人工终止或其他原因 |
| `Operation not permitted`（unshare/tc） | 当前系统不允许相应命名空间／网络操作 | 不是已经成功施加论文网络条件 |
| `GnuTLS recv error` / HTTP/2 clone 失败 | 下载连接异常 | 与实验算法正确性不是同一个问题 |

## 12. 大规模、线程和崩溃文件：本机遇到的问题

论文工作站是 Xeon Gold 6230R、52 核、128 GB RAM；本次笔记本及 WSL 的可用资源明显更少。大规模不应一开始就全部排队，先确认较小规模的流程、内存和日志。

### 12.1 APSI 单线程指什么

原始主程序使用：

```cpp
ThreadPoolMgr::SetThreadCount(std::thread::hardware_concurrency());
```

它依据程序看到的硬件并发数设置 APSI 工作线程；本次 WSL 配置为 4 个逻辑处理器时，默认是 4 个 APSI 工作线程。资源受限探索曾改成：

```cpp
ThreadPoolMgr::SetThreadCount(1);
```

它只调整 APSI 的工作线程池，不会把 BRPC、RR22、YACL 等所有线程都合并成一条。减少并行工作通常有助于降低同时处理的内存压力，但可能变慢，也不能保证大规模一定成功。

此外，本版 `aPSI.h` 的 `deleteItems` **原本就会临时将 APSI 删除操作限制为单线程**，再恢复原设置。所以不能说“改全局单线程必定修复删除异常”，也不能认为过去每一个步骤都用 4 个线程。

线程数改变后要重新构建并另存配置，不能把这组数据直接混入原来四线程条件的时间均值。

### 12.2 控制失败时的磁盘占用

本机曾出现 WSL 将大体积崩溃转储写入 Windows 临时目录的情况；仅 `ulimit -c 0` 在采用管道捕获转储的环境中不一定足够。

若需要在独立实验会话暂时禁用这种转储，可先记录现值，再设置：

```bash
cat /proc/sys/kernel/core_pattern
sudo sysctl -w kernel.core_pattern=core
ulimit -c 0
```

这会影响该 WSL 内核会话的崩溃转储行为，不是算法优化；应了解原设置，实验结束可恢复。WSL 重启后也要重新检查，不能认为这段命令永久生效。

可选的资源受限运行方式：

```bash
(
  ulimit -c 0
  ulimit -v 10485760

  ./run_benchmarks.sh \
    --backends=krtw \
    --scenarios=LAN \
    --repeats=1 \
    --update-count=1 \
    --link=brpc \
    --skip-build \
    --output-dir="/mnt/f/PSI-results/UPSI/resource_limited_$(date +%Y%m%d_%H%M%S)"
)
```

`ulimit -v` 的单位为 KiB，`10485760` 约为 **10 GiB 虚拟地址空间上限**，不是给程序分配 10 GiB RAM。它也可能主动导致分配失败，必须记录在实验条件中。圆括号建立子 Shell，避免这次上限长期影响后续终端命令。这段用于资源诊断，不作为默认论文性能测试设置。

本次 N=$2^{22}$、Nu=$2^{12}$ 的探索未得到可用的更新结果，后来按整理要求删除失败产物并在当前汇总中标为“未测”；Nu=$2^{13}$ 没有实际完成测试。这里的“未测”是当前结果表的展示规则，不表示历史上从未尝试过前一组。

## 13. 从头到尾的执行顺序清单

| 顺序 | 要做的事 | 完成后检查什么 |
|---:|---|---|
| 1 | PowerShell 安装、启动 WSL2 Ubuntu | `wsl -l -v` 显示版本 2 |
| 2 | Ubuntu 安装构建与网络工具 | `g++`、`cmake`、`tc` 可用 |
| 3 | 安装 Bazelisk，选择 Bazel 7.4.1 | `bazel --version` |
| 4 | clone 两个仓库并固定提交 | 两个 `git rev-parse HEAD` |
| 5 | 将 UPSI 放入 `yacl/examples/upsi` | `WORKSPACE`、`BUILD.bazel` 存在 |
| 6 | 确认 N、Nu、两处 JSON、线程策略 | `git diff` 和参数文件内容 |
| 7 | 在 YACL 根目录编译 | `Build completed successfully` |
| 8 | LAN 一次功能验证 | 退出码 0，初始和更新双方交集检查通过 |
| 9 | 同规模四网络各三次 | 12 次记录、成功数和失败数明确 |
| 10 | 记录程序实际规模、JSON 与网络配置 | 目录标签与实际参数一致 |
| 11 | 改下一规模并重新构建 | 不误用上一规模的旧二进制 |
| 12 | 用逐次记录汇总到两类表格 | 只对成功样本算均值，保留失败状态和JSON |

已经完成安装、克隆和编译的人，不必从第一条再跑。回到自己的实际进度，先核实当前源码与程序的配置，再继续对应步骤。

## 14. 进一步读源码，先看哪里

建议按照“实验控制 → 协议组合 → 底层组件”的顺序阅读：

1. **`run_benchmarks.sh`**：弄懂网络、重复次数和结果文件如何产生。
2. **`main.cc` 的 `RunUPSIv1`**：找到规模、JSON、计时与正确性检查。
3. **`upsi.cc` 的 `UPsiSendV1/UPsiRecvV1`**：观察 APSI 查询与两类 PSU 合并如何更新旧交集。
4. **`aPSI.h/aPSI.cc`**：看 SenderDB 的增删、OPRF、查询、响应与链路适配。
5. **`psu/`、`rr22/` 和 YACL**：在需要定位具体瓶颈或异常时深入。

先能回答“这一个数字由哪段计时代码产生”，再讨论优化哪个密码学组件，会比一开始逐行读完所有第三方库更容易建立完整理解。

## 参考资料与版本来源

- [论文：Efficient Updatable PSI from Asymmetric PSI and PSU](https://ieeexplore.ieee.org/document/11361182)，实验环境与统计见 VI-A、VI-C 及表 I、II。本文依据论文与本地实际源码解释，不重新分发论文 PDF。
- [作者 UPSI 固定提交](https://github.com/ShallMate/upsi/tree/e3dd3731f29c791d686c464f6fe48979db025130)：本文的 `main.cc`、`upsi.cc`、APSI 包装、参数文件和脚本依据。
- [作者 YACL 固定提交](https://github.com/ShallMate/yacl/tree/46d6137b297d9ac09fcb8401ddd89e705bcba175)：工作区、Bazel 构建配置和依赖声明。
- [APSI v0.11.0](https://github.com/microsoft/APSI/tree/v0.11.0)：与本次构建对应的接口与参数背景，避免混用新版本文档。
- [Bazel 官方介绍](https://bazel.build/about/intro)：构建工具、依赖与缓存的基本概念。
- [YACL 上游项目](https://github.com/secretflow/yacl)：库的定位和模块说明；实际实验仍以作者的固定分支提交为准。
- [Bazelisk v1.22.1](https://github.com/bazelbuild/bazelisk/releases/tag/v1.22.1)：文中启动器下载版本。
- [WSL 基础命令](https://learn.microsoft.com/en-us/windows/wsl/basic-commands)、[WSL 网络访问说明](https://learn.microsoft.com/en-us/windows/wsl/networking)：Windows 与 Linux 环境及网络配置。

文中安装和构建命令以列出的版本及 x86-64 环境为基准；规模、线程、参数、资源限制与网络条件应始终一起记录。后续若更换这些条件，应建立新的实验配置，而不是直接覆盖旧结果。

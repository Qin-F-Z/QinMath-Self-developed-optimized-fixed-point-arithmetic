# 秦数学 (QinMath) 算法总结与使用文档 V2

> **版本**: V2 (多级整数结构 + 6 种舍入模式 + REGISTER_CARRY 乘法 32 进制寄存累积机制 + 加减运算 + 比较运算 + 属性保护 + ops_log 自动 flush + 详细哈希)
> **适用对象**: 工程师、数学家、计算机科学家、审计人员
> **许可证**: 专利保护, 未授权使用侵权

---

## 目录

1. [概述](#1-概述)
2. [核心概念对照表](#2-核心概念对照表)
3. [六大核心机制（详解逻辑与运算方法）](#3-六大核心机制详解逻辑与运算方法)
4. [快速上手](#4-快速上手)
5. [API 参考](#5-api-参考)
6. [舍入模式使用指南](#6-舍入模式使用指南)
7. [已知限制](#7-已知限制)
8. [适用场景](#8-适用场景)
9. [附录: 设计哲学](#9-附录-设计哲学)

---

## 1. 概述

秦数学 (QinMath) 是一种**多级整数分离存储定点算术引擎**, 面向需要**确定性**和**可审计性**的科学计算场景。

### 核心理念

| 理念 | 含义 |
|---|---|
| **真值 = 行业常规标准** | R0 真值模式下 K+32 位精度范围内的表示, 不是"超越 K 位的理想真值"。R0 走同一套秦数学体系, 与 R1 在体系内比对, 不校正 R1 |
| **多级整数结构** | 整数无界(第0级) + 小数有界K位(第1级) + 暗藏register k=32位(第2级) |
| **乘法累积+满量进位** | REGISTER_CARRY 是乘法 32 进制寄存累积机制, register 满 10³² 时进位恢复到 decimal_part |
| **除法不累积** | 除法存最后余数到 register, 不累积 |
| **finalize 只做 K 边界舍入** | 最后一步一级 K 边界按 R1 舍入(register 影子决定), 不再做满量进位。**finalize 是正常计算的最后一步, 不是修正** |
| **运算中不修正** | 运算过程中(multiply/divide/sqrt/add/sub)不做 K 边界舍入, 余数只记录不修正 |
| **运算中叫 register (寄存)** | 运算过程中的余数只记录不修正 |
| **R0/R1 比对靠用户外部** | deviation = R1 结果 对比 R0 结果, 由用户/审计员在外部跑两遍比对, 引擎不自动生成, 不以 R0 校正 R1 (硬性规定) |
| **计算结果由 R1 决定** | 用户选择的舍入模式直接决定结果; R0 只是审计基准 |
| **跨平台 bit-identical** | Python/JS/Solidity/Rust 结果完全一致 (同顺序、同 R1、同 K 下) |

### 与传统固定位宽定点算术的区别

| 维度 | 传统固定位宽定点 | 秦数学 |
|---|---|---|
| 存储 | 对称固定位宽 | 非对称多级: 整数无界 + 小数K位 + register k=32 |
| 余数 | 直接丢弃 | 寄存 32 位 register (审计+乘法进位恢复) |
| 修正 | 无选择 | 运算中不修正; finalize 做 K 边界舍入 (正常计算最后一步) |
| 审计 | 无 | SHA-256 证据链 + 完整审计 (含 ops_log_fragments) |
| 跨平台 | 部分 | bit-identical (同顺序下) |
| 舍入模式 | 隐式或固定 | 6 种显式 + R0/R1 双模式系统 |
| 乘法累积 | 无 | REGISTER_CARRY: 32 进制寄存累积, 满量进位恢复 |

> **bit-identical 脚注**: "跨平台结果一致" 指**同程序按相同运算顺序、相同 R1 模式、相同 K 值**跑出的结果一致；不保证交换律、结合律在 register 层面成立 (详见 §6 舍入模式使用指南顶部声明)。

---

## 2. 核心概念对照表

### 表 1: 核心存储结构（多级整数结构）

| 级别 | 字段名 | 类型 | 取值范围 | 物理意义 | 备注 |
|---|---|---|---|---|---|
| 第0级 | `integer_part` | int (无界) | -∞ ~ +∞ | 整数部分 | 无上溢出 |
| 第1级 | `decimal_part` | int | [0, 10^K) | K位小数的整数化 | 有界K位, **不可变** |
| 第2级 | `register` | int | [0, 10^32) | 暗藏二级余数 | **固定k=32位**, 累积阶段始终非负 (5A 方案) |
| 配置 | `precision_k` | int | 默认18 | 小数保留位数 | 可配置, **不可变** |
| 派生 | `_scale` | int | = 10^K | 一级单位换算 | 自动派生 |
| 派生 | `_REGISTER_SCALE` | int | = 10^32 | 二级单位换算 | 固定, 概念常量 |
| 舍入标识 | `has_rounding` | bool | True/False | register≠0即True | 显示"!"前缀, **finalized 后不可改** |
| 封存状态 | `finalized` | bool | True/False | 是否已封存到证据链 | finalize 后置 True, **不可改** |
| 审计 | `_ops_log_fragments` | list | 文件路径列表 | 已 flush 的外部审计文件 | 纳入 evidence_hash (8A 方案) |

> **多级整数结构**: 秦数学本质是多级整数运算, 不是简单的 K 位定点。
> - 第0级整数无界, 支持 10^77+ 不溢出
> - 第1级小数有界 K 位, 是用户可见的精度
> - 第2级 register 暗藏 k=32 位, 是 sub-ULP 的影子, 用于乘法累积进位和 finalize 舍入判断
> - 级间满量进位: register 满 10^32 → decimal_part; decimal_part 满 10^K → integer_part

**多级整数结构线性示意图** (从低位到高位, 箭头表示满量进位方向):

```
  低位                                                    高位
  ┌──────────────┐      满10^32      ┌──────────────┐      满10^K      ┌──────────────┐
  │   register   │ ────进位恢复────→ │ decimal_part │ ────进位──────→ │ integer_part │
  │   第2级      │     (乘法累积)    │   第1级      │                 │   第0级      │
  │  k=32 暗藏   │                  │  K位 小数    │                 │  无界 整数   │
  └──────────────┘                  └──────────────┘                 └──────────────┘
        │                                  │                                  │
   sub-ULP影子                       用户可见精度                        整数部分
  (finalize用影子                    (R1舍入的K边界)                    (无上溢出)
   决定+1方向)
```

> **读图说明**:
> - **从左到右**: 从低位(register)到高位(integer_part), 数据流动方向
> - **箭头**: 表示满量进位 — register 满 10^32 进位到 decimal_part, decimal_part 满 10^K 进位到 integer_part
> - **进位是无损的 (在二级 32 位范围内)**: 整数除法+取模, 信息不丢失 (见机制2)
> - **超过二级 32 位外的 sub-sub-ULP 会被截断**: 这是定点算术的固有限制, 如同 IEEE 754 的 subnormal 下溢
> - **R1 舍入只在 finalize**: 一级 K 边界用 register 影子决定是否 +1 (有损, 见表8)
> - **register 暗藏**: 用户不可见, 仅用于审计+乘法累积进位+finalize 影子

### 表 2: 舍入模式 (RoundingMode) — 6 种显式模式

| 模式 | 枚举值 | 正数行为 | 负数行为 | 适用场景 |
|---|---|---|---|---|
| 向下 | `DOWN` | 截断 | 远离零 | 金融保守（不高估） |
| 向上 | `UP` | 远离零 | 截断 | 上限估计（保守偏大） |
| 向零 | `TOWARD_ZERO` | 截断 | 截断 | 物理截断（快速近似） |
| 远离零 | `AWAY_FROM_ZERO` | 远离零 | 远离零 | 安全余量（材料/核能） |
| 四舍五入 | `HALF_UP` | r×2≥d进位 | r×2≥d进位 | 通用计算 |
| 银行家 | `HALF_EVEN` | 半数向偶 | 半数向偶 | 统计/金融（无偏） |

> **REGISTER_CARRY 不是舍入模式, 而是乘法 32 进制寄存累积机制**:
>
> - 乘法运算中, sub-ULP 累积到 register
> - register 满 10^32 时, 进位恢复到 decimal_part (满量进位)
> - 这是乘法的**固有累积行为**, 与 R1 舍入模式无关
> - R1 舍入模式只决定最后一步 finalize 时 K 边界怎么舍入(register 影子决定)
> - REGISTER_CARRY 不是 RoundingMode 枚举的一员, 不需要用户选择, 乘法默认总是开

### 表 3: 双模式系统

| 符号 | 字段名 | 含义 | 设定方式 | 可变性 |
|---|---|---|---|---|
| **R0** | `truth_mode` | 真值模式（行业常规标准, 审计基准） | 全局初始化时设定 | 链式运算中**不可变** (运行时 `__setattr__` 拦截抛 AttributeError) |
| **R1** | `rounding` | 计算模式（用户自选舍入方式） | 每次运算传入 | **每次运算可变** |

> R0 只用于定义"真值基准"供最终审计比对, **不约束**用户每次运算的 R1 选择。
> 用户选不同 R1 才有不同结果——这正是 R1 存在的意义。

> **R0 与 R1 走同一套秦数学体系**:
> - **秦数学 R0 模式** = 用 R0 当作 R1 跑一遍运算 (R0 既是 truth_mode 也是 rounding)
> - **秦数学 R1 模式** = 用 R1 跑一遍运算 (R0 是 truth_mode, R1 是 rounding)
> - deviation = R1 结果 对比 R0 结果 (两者都是秦数学跑出来的, 用户外部比对)
> - **不以 R0 标准自动校正 R1** (硬性规定, 引擎不提供自动 deviation 计算)

### 表 4: 余数寄存 (register) — 单 register 设计 (5A 方案)

> **命名约定**: 运算过程中的余数存储叫 **register (寄存)**, 中性词, 只记录不修正。
> 统一存储到 `register` 字段 (固定 k=32 位), **不分** mul_register/div_register。

> **5A 方案: register 在累积阶段始终非负**:
> - 乘法用**绝对值运算**, register 始终非负
> - 数字符号由 `integer_part` / `decimal_part` 承载
> - 满量进位规则仅处理非负 register: `register ≥ 10^32` 时进位恢复
> - 负 register 只在 finalize 后出现 (表示超计, 审计用), 不参与累积
> - 避免乘法累积时符号混乱 (与"除法不累积避免符号反转"一致)

| 运算 | 累积行为 | 满量进位? | register 用途 |
|---|---|---|---|
| 乘法 | **累积** (保留链式累积) | **是** — register 满 10^32 时进位恢复到 decimal_part | 累积+进位恢复+审计 |
| 除法 | **不累积** (覆盖, 存最后余数) | **否** — 仅审计+finalize 影子, 不进位 | 存最后余数+审计 |
| sqrt | **不累积** (覆盖, 存最后余数) | **否** — 仅审计+finalize 影子, 不进位 | 同除法 |
| 加法 | **不产生新 sub-ULP**, register 保留原样 | 否 | 之前 register 保留 (不抹掉) |
| 减法 | **不产生新 sub-ULP**, register 保留原样 | 否 | 之前 register 保留 (不抹掉) |

> **为什么单 register (不分区 mul_register/div_register)**:
> - **当前值只有一个 sub-ULP 状态**: 每步运算后, QinNumber 代表"当前值", 当前值只有一个 sub-ULP 影子, 用单 register 即可表达
> - **混合运算 a×b÷c 时**: 乘法累积的 register 会被除法覆盖, 但:
>   - 乘法满量部分(≥10^32)已在运算中进位恢复到 decimal_part (REGISTER_CARRY)
>   - 被覆盖的只是未满量的 sub-ULP (< 1 个 decimal ULP, 在 K 位精度外)
>   - ops_log 完整记录了每步的 register (审计可追溯)
> - **分区存储的代价大于收益**: 分区需额外字段+finalize 合并策略(两个影子怎么决定 K 边界?), 复杂度增加但只换来精度外 sub-ULP 的保留

> **关键**:
> - 乘法: register 累积 sub-ULP, 满量进位恢复精度 (REGISTER_CARRY 机制, **在规则内恢复精度**)
> - 除法/sqrt: register 不累积, 覆盖存最后余数, 用于 finalize 影子+审计
> - 加减: 不产生新 sub-ULP, register 保留原样 (不抹掉)
> - **deviation** 由用户外部比对 R0/R1 结果得出, 引擎不自动生成 (见 §3 硬性规定)
> - register 固定 k=32 位, 跨平台 bit-identical

### 表 5: R0 vs R1 行为矩阵

| 场景 | register 存在? | "!"标识 | deviation | 说明 |
|---|---|---|---|---|
| R1 运算, register=0 | 无 | 无 | 无 | 无 sub-ULP, 未舍入 |
| R1 运算, register≠0 | 有 | 有(!) | 暂无 | 有 sub-ULP, register 已记录 |
| 用户外部比对 | — | — | 用户得出 | deviation = R1 结果 对比 R0 结果 (用户外部跑两遍比对, 引擎不自动生成) |

> 注意: 运算阶段只产生 register, 不产生 deviation。deviation 是用户/审计员在外部把 R1 结果拿去和 R0 结果比对的产物, 引擎不自动生成, 不以 R0 校正 R1 (硬性规定)。

### 表 6: 符号与显示

| 符号 | 含义 | 示例 |
|---|---|---|
| 无前缀 | 未舍入(register=0), 无 sub-ULP | `0.125` |
| `!` 前缀 | 已舍入(register≠0), 有 register 记录 | `!0.056` |
| `Q(a.b \| ...)` | 调试格式 | `Q(0.056 \| reg=88 R0=half_even R1=half_even)` |

> **重要**: 无前缀只是"这次运算没产生 sub-ULP/没舍入"的结果, **不等于 R0 真值**。
> `!` 仅表示"这步发生了舍入", 不表示"偏离了真值"。

### 表 7: 真值定义

> **真值 = 行业常规标准。** 不是某种脱离秦数学体系的"绝对真值", 而是 R0 模式下 K+32 位精度范围内的工程约定。
> R0 走同一套秦数学体系 (多级整数、register、REGISTER_CARRY 等), 与 R1 在体系内比对, 不校正 R1。
> R0 真值不是"圣经"——用户选不同 R1 模式就是为了得到不同结果, 每种 R1 结果都是合法的计算结果。

### 表 8: 两层 K 边界处理（乘法二级满量进位 / 最后一步计算一级 R1 舍入）

> **两层边界处理设计** — 为什么不能两层都做 R1 舍入:
> - **二级 K 边界 (register≥10^32)**: 做**满量进位**(有条件无损), **不消耗** register 影子
>   - 整数除法+取模: `carry = register // 10^32; register %= 10^32; decimal_part += carry`
>   - 信息末端丢失，超过二级 32 位的数字被丢弃截断, 32 位的数字register 影子完整保留给 finalize
> - **一级 K 边界 (finalize)**: 做 **R1 舍入**(有损, 用 register 影子决定)
>   - 看 register 相对 10^32 的比例, 按 R1 决定是否给 decimal_part +1
>   - 这是唯一消耗 register 影子的地方
>   - **finalize 是正常计算的最后一步 K 边界舍入, 不是对之前结果的修正**
> - **两层 R1 舍入不可行**: 二级 K 边界 R1 舍入会提前消耗/混乱 register 影子, finalize 失去完整判断依据, 精度下降

| 阶段 | 边界 | 处理方式 | 消耗影子? | 有损? |
|---|---|---|---|---|
| 运算中(乘法累积) | 二级 K(register≥10^32) | 满量进位(整数 carry 到 decimal_part) | 否 | 有损, 二级 32 位内 |
| 运算中(除法) | 二级 K(register<10^32, 不超界) | 不处理(单步映射值<10^32) | 否 | 有损 |
| finalize | 一级 K(decimal_part) | R1 舍入(register 影子决定 +1) | 是 | 有损 |

> **finalize 的 K 边界舍入逻辑**:
> - 看 register 影子 (register 相对于 10^32 的比例)
> - 按 R1 舍入模式决定是否给 decimal_part 最后一位 +1:
>   - TOWARD_ZERO: 不 +1, register 保留(欠计)
>   - AWAY_FROM_ZERO: +1, register 调整(超计)
>   - DOWN: 正数不 +1, 负数 +1
>   - UP: 正数 +1, 负数不 +1
>   - HALF_UP: register×2 ≥ 10^32 则 +1
>   - HALF_EVEN: register×2 > 10^32 则 +1, 精确半数看 decimal_part 最后一位奇偶
> - register 舍入后可能变负(表示超计), 仅审计用
> - 正规化: +1 后若 decimal_part≥scale, 进位到 integer_part
> - 封存到证据链: 完整审计信息写入 canonical JSON, 标记 finalized=True

### 表 9: 证据链 (evidence_hash) — 8A 方案

| 字段 | 内容 | 用途 |
|---|---|---|
| `integer_part` | 整数部分 | 结果验证 |
| `decimal_part` | 小数部分(zfill K 位) | 结果验证 |
| `precision_k` | K 值 | 配置复现 |
| `register_k` | 32 (固定) | 配置复现 |
| `truth_mode` | R0 模式 | 真值基准 |
| `rounding_used` | R1 模式 | 运算意图 |
| `has_rounding` | 是否舍入 | "!"标识来源 |
| `register` | register 值 | 审计 |
| `ops_count` | 运算次数 | 链长 |
| `ops_log` | 完整运算日志 (最近 10 条) | 第三方复现 |
| `ops_log_fragments` | 已 flush 的外部文件路径列表 | 跨文档审计链完整性 (8A 方案) |
| `finalized` | 是否封存 | 状态标识 |

> 审计哈希: `_ops_log_fragments` 纳入 evidence_hash 计算, 任何外部审计文件被替换/丢失/重命名, evidence_hash 都会变化, 保证跨文档审计链完整性。
> 单 register 被覆盖也不影响审计完整性——哈希绑定全部 ops_log + fragments, 审计员可从 ops_log 重建历史 sub-ULP 状态。

---

## 3. 六大核心机制（详解逻辑与运算方法）

> 本节是秦数学的核心, 详细描述每个机制的**运算逻辑**与**实现方法**, 供工程师直接实现、机器直接执行。

### 机制 1: 多级整数分离存储

**运算逻辑**:
- 把一个数 N 拆成多级整数: `N = integer_part + decimal_part/scale`
- 第0级 `integer_part`: 无界整数 (Python int / 硬件 uint256 扩展), 永不溢出
- 第1级 `decimal_part`: 有界 K 位, 范围 `[0, 10^K)`, 即最多 K 位十进制小数
- 第2级 `register`: 暗藏 k=32 位, 范围 `[0, 10^32)`, sub-ULP 的影子
- `scale = 10^K` 是一级单位换算, `_REGISTER_SCALE = 10^32` 是二级单位换算

**实现方法**:
- `integer_part`: Python int (无界), 硬件实现可用 uint256 + 进位传播
- `decimal_part`: K 位 int, 范围 `[0, 10^K)`
- `register`: 32 位 int, 范围 `[0, 10^32)`, 固定不可配置
- `__post_init__` 自动派生 `_scale = 10 ** precision_k`

**运算示例** (K=18):
- π ≈ 3.14159... → `integer_part=3, decimal_part=141592653589793238, register=0, scale=10^18`
- 10^77 量级的大数: `integer_part=10^77, decimal_part=0, register=0` — 无溢出

**优势**: 适配非对称精度需求 (如金融: 10^77 资产 + 10^-18 精度共存), 暗藏 32 位 register 提供 sub-ULP 精度

---

### 机制 2: 大值分区处理（级间满量进位 — 二级无损 / 一级 R1 舍入）

**运算逻辑**:
- 一级 K 边界: 当 `decimal_part ≥ scale` 时, 进位到 `integer_part`
- 一级借位: 当 `decimal_part < 0` 时, 从 `integer_part` 借位
- 二级 register 边界: 当 `register ≥ 10^32` 时, **满量进位**(无条件, 无损)恢复到 `decimal_part` (乘法累积)

**两层边界处理的关键区别**:
- **二级 K 边界 (运算中)**: 满量进位 — 整数除法+取模, **无损** (二级 32 位范围内), **不消耗** register 影子
- **一级 K 边界 (finalize)**: R1 舍入 — 看 register 影子按 R1 决定 +1, **有损**, **消耗**影子
- 为什么二级 K 不做 R1 舍入: 会提前消耗/混乱 register 影子, finalize 失去完整判断依据 (见表8详解)

**实现方法**:
- `_normalize_decimal()`:
  - 若 `decimal_part >= scale`: `carry = decimal_part // scale; decimal_part %= scale; integer_part += carry`
  - 若 `decimal_part < 0`: `borrow = (-decimal_part + scale - 1) // scale; decimal_part += borrow * scale; integer_part -= borrow`
- `_register_carry()` (乘法累积满量进位, **无损**, 仅处理非负 register, 5A 方案):
  - 若 `register >= _REGISTER_SCALE`: `carry = register // 10^32; register %= 10^32; decimal_part += carry`
  - 整数除法+取模, 信息不丢失 (二级 32 位范围内), register 影子完整保留给 finalize
  - 这是 REGISTER_CARRY 机制的核心实现

> **"满量进位" 的边界**:
> 秦数学的进位规则是标准十进制进位, 在归一化阶段执行。register 满 10^32 = 1 个 decimal ULP, 进位恢复丢弃值。
> 与 R1 舍入不同, 满量进位是无条件无损的 (在二级 32 位范围内)。
> **超过二级 32 位外的 sub-sub-ULP 会被截断** — 这是定点算术的固有限制, 如同 IEEE 754 的 subnormal 下溢, 无法避免 (无限延伸三级、四级 register 不可行)。

---

### 机制 2.5: 加减法（不产生 sub-ULP, register 保留）

**运算逻辑**:
- 加减是纯 L0+L1 运算:
  - `integer_part + integer_part` → 整数部分
  - `decimal_part + decimal_part` → 小数部分 (最多向 integer_part 进位)
- **不产生新的 sub-ULP** (两个 K 位小数相加, 结果还是 K 位小数, 不会触发 register)
- **register 保留原样** (不抹掉、不累积、不归零): 之前乘法留下的 register 不被加减抹掉, 否则审计断裂

**实现方法**:
- `add(other)`:
  - 若 self 已 finalized: 抛弃影子(register=0), 按账面 K 位重新开始新链 (见机制 4 finalize 重启)
  - 否则: `result.integer_part = self.integer_part + other.integer_part`
  - `result.decimal_part = self.decimal_part + other.decimal_part`
  - `result.register = self.register` (保留, 不动)
  - `result.has_rounding = (result.register != 0) or self.has_rounding`
  - `_normalize_decimal()` 处理 decimal_part 溢出
- `sub(other)`: 调用 `add(other 的相反数)`

**register 保留验证**:
```python
x = QinNumber.from_string('3.141592653589793238')
y = QinNumber.from_string('2.718281828459045235')
mul = x.multiply(y)                         # register ≠ 0
z = QinNumber.from_string('100')
add_after_mul = mul.add(z)                  # 加法不抹掉 register
assert add_after_mul.register == mul.register  # ✅ 保留
```

---

### 机制 3: 乘除余数寄存（乘法累积 / 除法不累积, 5A 方案）

**运算逻辑**:
- **乘法 (累积+满量进位, 5A 方案)**:
  - 用**绝对值运算**, register 始终非负
  - sub-ULP 映射到 register 并累积 (保留链式累积)
  - register 满 10^32 时进位恢复到 decimal_part (REGISTER_CARRY)
  - 运算中不做 K 边界舍入, 只做整数倍进位
  - 数字符号由 integer_part / decimal_part 承载
- **除法 (不累积+存最后)**:
  - 余数映射到 register, 不累积 (覆盖, 存最后)
  - 超过 10^32 边界按 R1 舍入 (运算中只做整数倍进位)
- **sqrt (同除法)**: 不累积, 存最后余数

**sub-ULP 映射方法**:
```
# 乘法: sub_ulp = a_dec × b_dec % scale
register_val = sub_ulp × 10^32 // scale  (保持比例, 暗藏 32 位精度)

# 除法: r 是余数, d 是除数绝对值
register_val = r × 10^32 // d  (映射到 [0, 10^32))
```

**除法寄存的价值 **:
- 除法确实寄存余数到 register, 不是丢弃
- register 的两个作用:
  1. **finalize 影子**: 最后一步用 register 影子决定 K 边界怎么舍入 (没有 register 就无法判断 sub-ULP 方向)
  2. **审计记录**: ops_log 完整记录每步 register, 可追溯
- **大数值单位下的决定性作用**: K=8 (以亿为单位) 时, 10 亿÷3 的 register 记录的 sub-ULP 对精度有决定性影响
- **长链除法**: 中间步骤 register 被下一步覆盖 (只存最后), 但 ops_log 记录了每步, 丢失的是精度外 sub-ULP

**关键原则 **:
- 运算中叫 **register** (寄存), 不叫 deviation
- 乘法 register 累积满量会进位 (REGISTER_CARRY), 这是乘法特有的精度恢复机制 (**在规则内恢复精度**)
- 除法/sqrt register 不进位, 用于 finalize 影子+审计
- 加减不产生新 sub-ULP, register 保留原样
- **负 register 不参与累积**: 仅 finalize 后出现 (表示超计, 审计用), 与"除法不累积避免符号反转"一致
- **deviation** 由用户外部比对 R0/R1 结果得出, 引擎不自动生成 (见硬性规定)
- register 固定 k=32 位, 跨平台 bit-identical

---

### 机制 4: 分层定点运算调度（乘法累积+满量进位 / 除法不累积+存最后 / finalize 重启）

**运算逻辑** — 为什么乘法累积、除法不累积:

- **乘法 (累积 + REGISTER_CARRY 满量进位, 5A 方案)**:
  - 乘法余数方向一致 (同号累积), 可安全累积到 register
  - 用绝对值运算, register 始终非负 (5A 方案)
  - register 满 10^32 时进位恢复到 decimal_part (REGISTER_CARRY 机制)
  - 这是乘法的固有 32 进制寄存累积行为, 与 R1 舍入模式无关
- **除法 (不累积 + 存最后)**:
  - 除法余数方向**不一致**(取决于分子分母符号与舍入方向), 累积会导致正负相消或符号反转
  - 因此只存**最后一次**余数到 register, 不累积, 也不进位 (仅审计)
- **sqrt (同除法)**: 不累积, 存最后余数
- **加减 (不产生新 sub-ULP, register 保留)**: 见机制 2.5

**实现方法**:
- 乘法累积: `result.register = prev_register + sub_ulp_mapped` (累积, 非负)
- 除法: `result.register = sub_ulp_mapped` (覆盖, 不累积)
- 加减: `result.register = self.register` (保留, 不动)
- register 满量进位 (乘法运算中, 仅处理非负 register):
```
if register >= _REGISTER_SCALE:
    carry = register // _REGISTER_SCALE
    register %= _REGISTER_SCALE
    decimal_part += carry  # 进位恢复
    _normalize_decimal()  # 处理 decimal_part 溢出
```

**设计原因**:
- 除法不累积避免了 BUG-A (除法链符号反转) — UP/AWAY_FROM_ZERO 模式在长链除法中累积负余数会导致结果符号反转, 不累积策略天然避免此问题
- 乘法累积+满量进位是 REGISTER_CARRY 的核心价值: **在秦数学规则内** (乘法累积), 单步截断的余数不丢弃, 累积后进位恢复精度; 混合运算中未满量部分会被除法覆盖

**finalize 后重启新链**:
- finalized=True 后调用 `multiply / divide / sqrt / add / sub` 时:
  1. 隐式 `copy()` 一份副本
  2. 旧 `register` **清零抛弃** (影子已在 finalize 消耗, 用于 K 边界舍入 + 审计)
  3. `finalized=False`、`ops_count=0`、`_ops_log=[]` (新链开始)
  4. 按"当前账面 K 位"为新基线开始新一轮运算

---

### 机制 5: 显式舍入模式（6 种 + R0/R1 双模式系统）

**运算逻辑**:
- 6 种显式舍入模式: DOWN / UP / TOWARD_ZERO / AWAY_FROM_ZERO / HALF_UP / HALF_EVEN
- **REGISTER_CARRY 不是舍入模式**, 而是乘法 32 进制寄存累积机制 (乘法默认总是开)
- **双模式系统**:
  - R0 (`truth_mode`): 全局不可变 (`__setattr__` 拦截), 定义"真值基准"供审计比对
  - R1 (`rounding`): 每次运算可变, 用户自选——**这才是决定计算结果的模式**
- R0 不约束 R1: 用户每次运算可选不同 R1, 得到不同结果, 每种都合法
- R1 只决定**最后一步 finalize** 时 K 边界怎么舍入 (register 影子决定)

**实现方法**:
- `_round_k_boundary(decimal_part, register, sign, mode)`:
  在一级 K 边界按 R1 舍入, register 影子决定方向。返回 `(new_decimal_part, new_register)`:
  - `TOWARD_ZERO`: 不 +1, register 保留(欠计)
  - `AWAY_FROM_ZERO`: +1, register 调整(超计)
  - `DOWN`: 正数不 +1, 负数 +1
  - `UP`: 正数 +1, 负数不 +1
  - `HALF_UP`: `register×2 ≥ 10^32` 则 +1
  - `HALF_EVEN`: `register×2` 与 `10^32` 比较, 半数向偶数取整
- 调用方: `multiply(other, rounding=None)` / `divide(other, rounding=None)` / `sqrt(rounding=None)`
  - `rm = rounding or self.rounding` (R1, 每次可变)
  - finalize 时调用 `_round_k_boundary`

**符号约定 (5A 方案)**:
- 累积阶段: `register` 始终非负 (乘法用绝对值运算)
- finalize 后: `register` 可能变负 (表示超计, 审计用)
- `register = 0`: 无 sub-ULP, 无舍入

---

### 机制 6: 证据链绑定（canonical JSON + SHA-256, 8A 方案）

**运算逻辑**:
- 把计算结果的完整状态序列化为**确定性 JSON** (canonical, 键排序)
- 对 JSON 做 SHA-256 哈希, 生成证据指纹
- 任何字段篡改都会改变哈希, 实现可验证审计

**实现方法 (8A 方案, 详细哈希)**:
- `to_canonical_json()`:
  - 序列化字段: `integer_part`, `decimal_part` (zfill K 位), `precision_k`, `register_k` (32), `truth_mode` (R0), `rounding_used` (R1), `has_rounding`, `register`, `ops_count`, `finalized`, `ops_log[-10:]`, **`ops_log_fragments`** (已 flush 的外部文件路径列表)
  - `sort_keys=True, ensure_ascii=False` 保证跨平台确定性
  - 8A 方案: `_ops_log_fragments` 纳入哈希计算, 任何外部审计文件被替换/丢失/重命名, evidence_hash 都会变化
- `evidence_hash()`: `sha256(canonical_json).hexdigest()`
- `verify(expected_hash)`: 重算哈希并比对, 检测篡改

**审计完整性保证链**:
| 层 | 机制 | 保证 |
|---|---|---|
| 1 | canonical JSON 含 ops_log[-10:] | 最近 10 步运算可追溯 |
| 2 | canonical JSON 含 _ops_log_fragments (8A) | 任何外部审计文件替换/丢失都被检测 |
| 3 | evidence_hash = SHA-256(canonical JSON) | 任何字段篡改都改变 hash |
| 4 | verify(expected_hash) | 第三方可验证篡改 |

> 单 register 被覆盖也不影响审计完整性——哈希绑定全部 ops_log + fragments, 审计员可从 ops_log 重建历史 sub-ULP 状态, **审计完整性靠哈希绑定 ops_log, 不靠 register 完整保留**。

**第三方验证流程**:
1. 拿到结果对象 + 证据哈希
2. 调用 `verify(hash)` 重算比对 → True=未篡改, False=已篡改
3. 或独立用同规格实现重新计算, 比对哈希是否 bit-identical

> **反序列化约束声明**: `evidence_hash()` 用于验证篡改、`verify()` 用于比对。完整重建 QinNumber 对象并继续运算**必须用到秦数学引擎本身**, 第三方仅能验哈希、不能脱离秦数学重建内部状态。这是秦数学产品的商业护城河之一: 未来随秦数学计算软件发行配套的重建工具 SDK。

**ops_log 自动 flush 机制**:
- 阈值自动 flush: 对象内 `_ops_log` 满 `OPS_LOG_FLUSH_THRESHOLD` (默认 1000) 条后, 自动 flush 到外部文件
- flush 路径: 默认 `./qin_ops_logs/`, 可通过 `_flush_ops_log(out_dir)` 手动指定
- 文件名格式: `qin_ops_log_{chain_anchor前8位}_{fragment_index:03d}.json`
  - `chain_anchor` = flush 时刻的 `evidence_hash()[:8]`, 锚定计算链, 跨文档可追溯
- 外部文件 JSON schema:
  ```json
  {
    "chain_anchor": "a1b2c3d4",
    "fragment_index": 0,
    "ops_count_at_flush": 1000,
    "entries": [ /* 1000 条 ops_log 条目 */ ]
  }
  ```
- **跨文档审计**: 审计员按 `_ops_log_fragments` 列表顺序调入处理 (用户自行按顺序调入)

**ops_log 是单链日志, 不跨链拼接**:
- ops_log 是**单次链式运算**的日志, 链一结束就 finalize + 封存
- 不存在"跨链"的 ops_log 拼接需求
- 第 N 条动手脚的检测靠 `verify(hash)` — 任何字段改了哈希就变, 不需要 ops_log 追溯链外历史

---

## 4. 快速上手

### 安装与导入

```python
from qin_math_engine import QinNumber, RoundingMode, qin_sum
```

> **重要**: 不再依赖 `Decimal` 包装。所有构造入口最终都走 `from_string` 的纯字符串解析, 避免"其他整数算法"与秦数学逻辑打架。

### 基本运算

```python
# 初始化时设定 R0 真值模式 (全局不可变, __setattr__ 拦截)
# 推荐: 用 from_string 直接字符串构造, 省去 Decimal 包装
a = QinNumber.from_string('3.141592653589793238', k=18,
                          truth_mode=RoundingMode.HALF_EVEN)
b = QinNumber.from_string('2.718281828459045235', k=18,
                          truth_mode=RoundingMode.HALF_EVEN)

# 计算时用 R1 模式 (每次可变)
c = a.multiply(b, rounding=RoundingMode.HALF_UP)
print(c)  # !8.539734222673565678 ("!" 表示这步有 sub-ULP)

# 封存到证据链 (K 边界按 R1 舍入, register 影子决定)
c_final = c.finalize()
print(c_final.evidence_hash())  # SHA-256 hash
```

### 加减法与 qin_sum

```python
# 加法: 不产生 sub-ULP, register 保留原样
a = QinNumber.from_string('100.5', k=18)
b = QinNumber.from_string('200.25', k=18)
r_add = a.add(b)
print(r_add)  # 300.750000000000000000

# 减法
r_sub = a.sub(b)
print(r_sub)  # -100.250000000000000000

# 链式求和
vals = [QinNumber.from_string('100.5', k=18),
        QinNumber.from_string('200.25', k=18),
        QinNumber.from_string('300.125', k=18)]
r_sum = qin_sum(vals)
print(r_sum)  # 600.875000000000000000
```

### 比较运算 (K+32 位精度, 带 register)

```python
p = QinNumber.from_string('1.0', k=18)
q = QinNumber.from_string('0.9999999999999999999999999999', k=18)  # 小数28位→超长进 register
print(p > q)   # True  (register 让 q 比 1.0 小一点点)
print(p == q)  # False (K+32 位不等)

# K 值不同不能比较 (维度不同)
q2 = QinNumber.from_string('1.0', k=30)
# p == q2  → ValueError: K值不同的 QinNumber 不能比较 (维度不同)
```

### 指定舍入模式

```python
# R1 = AWAY_FROM_ZERO (安全余量, 适合材料学/核能)
a = QinNumber.from_int(10, k=18)
b = QinNumber.from_int(3, k=18)
r = a.divide(b, rounding=RoundingMode.AWAY_FROM_ZERO).finalize()
print(r)  # !3.333333333333333334 (进位, 远离零)
```

### 链式运算

```python
from qin_math_engine import qin_chain_multiply

# 链式乘法 (用 from_string 构造, 避免 float 运算结果的不精确)
values = [QinNumber.from_string('1.' + str(i).zfill(7), k=18)
          for i in range(100)]
result = qin_chain_multiply(values, rounding=RoundingMode.HALF_EVEN).finalize()
print(result)  # !1.000495120890162370
print(f"register (审计): {result.register}")
```

### 开平方

```python
# sqrt (同除法机制: 不累积, 存最后余数)
s = QinNumber.from_string('2.0', k=18, rounding=RoundingMode.HALF_UP)
sq = s.sqrt(rounding=RoundingMode.HALF_UP).finalize()
print(sq)  # !1.414213562373095049
```

### 大整数与超长小数

```python
# from_int: 大整数资产直接入秦数学
big = QinNumber.from_int(10**77, k=18)  # 10^77 资产, 无溢出
print(big.integer_part)  # 10^77 (78 位整数)

# from_string 超长小数 → 进 sub-ULP (不截断丢失)
pi_str = '3.1415926535897932384626433832795028841971'
pi_k5 = QinNumber.from_string(pi_str, k=5)
print(pi_k5.decimal_part)  # 14159 (前 5 位)
print(pi_k5.register != 0) # True (超长进入 sub-ULP, 不丢失)
# 这就是 sub-ULP 存在的意义: 大单位的小数不丢失精度
```

### 证据链验证

```python
# 生成证据
a = QinNumber.from_string('3.141592653589793238')
b = QinNumber.from_string('2.718281828459045235')
c = a.multiply(b).finalize()
hash_value = c.evidence_hash()

# 第三方验证
print(c.verify(hash_value))  # True

# 篡改检测 (finalized 后字段不可改, 必须用 object.__setattr__ 绕过才能演示)
import object
object.__setattr__(c, 'decimal_part', c.decimal_part + 1)  # 绕过保护做演示
print(c.verify(hash_value))  # False (篡改被检测)
```

### 构造方法对比

| 方法 | 输入 | 精度 | 适用场景 |
|---|---|---|---|
| `from_int(value, k=18)` | int 整数 | 精确 | 大整数资产、整数运算结果 |
| `from_string(s, k=18)` | 字符串 | 精确 (超长小数进 sub-ULP) | **推荐**, 高精度场景, 无理数构造 |
| `from_float(val, k=18)` | float 字面量 | 精确 (经 `from_string(str(val))`) | 简便, 仅限 float 字面量 |
| `from_decimal(val, k=18)` | Decimal | 精确 (经 `from_string(val.to_eng_string())`) | 兼容旧代码, 不依赖 Decimal*scale |

> **重要**: `from_float` / `from_decimal` 内部都改走 `from_string` 字符串解析, 不依赖 `Decimal * scale` 运算, 避免其他整数算法逻辑打架。
> `from_string` 超长小数进 sub-ULP (映射到 register), **不截断丢失** — 这是 sub-ULP 存在的意义, 大单位的小数不丢失精度。

---

## 5. API 参考

### QinNumber 类

#### 构造方法

```python
QinNumber(
    integer_part: int = 0,
    decimal_part: int = 0,
    precision_k: int = 18,
    truth_mode: RoundingMode = RoundingMode.HALF_EVEN,  # R0
    rounding: RoundingMode = RoundingMode.HALF_EVEN,   # R1
)
```

#### 类方法

| 方法 | 说明 | 推荐 |
|---|---|---|
| `from_int(value, k=18, ...)` | 从整数创建, `integer_part=value, decimal_part=0, register=0` | 大整数资产 |
| `from_string(s, k=18, ...)` | 从字符串解析, 超长小数进 sub-ULP (不截断丢失) | **推荐**, 高精度场景 |
| `from_float(val, k=18, ...)` | 从 float 创建 (经 `from_string(str(val))`, 不依赖 Decimal) | 仅限 float 字面量 |
| `from_decimal(val, k=18, ...)` | 从 Decimal 创建 (经 `from_string(val.to_eng_string())`, 不依赖 Decimal*scale) | 兼容旧代码 |

> **from_string register 符号规则 (5A 方案)**: register 始终非负 (是绝对值的 sub-ULP 影子), 数字符号由 integer_part / decimal_part 承载, 仅 finalize 后 register 可能变负 (表示超计, 审计用)。

#### 运算方法

| 方法 | 说明 |
|---|---|
| `add(other)` | 加法, 不产生 sub-ULP, register 保留原样 |
| `sub(other)` | 减法, 机制同加法 |
| `multiply(other, rounding=None)` | 乘法, register 累积+满量进位 (REGISTER_CARRY), 5A 方案绝对值运算 |
| `divide(other, rounding=None)` | 除法, register 不累积, 存最后余数 |
| `sqrt(rounding=None)` | 开平方, 同除法机制 |
| `finalize()` | 最后一步 K 边界按 R1 舍入 (register 影子), 封存证据链, 是正常计算的最后一步 (不是修正) |

#### 比较运算 (K+32 位精度, 带 register)

| 运算符 | 说明 |
|---|---|
| `a == b` | 数值相等比较 (只比数值, 不要求 ops_count/ops_log 相同), K 不同抛 ValueError |
| `a != b` | 由 `==` 自动派生 |
| `a < b` / `a <= b` / `a > b` / `a >= b` | K+32 位严格比较, K 不同抛 ValueError |
| `hash(a)` | QinNumber 可放入 set / dict 键 (与 `==` 一致) |

> 比较基准公式: `sign × (|integer_part| × 10^(K+32) + |decimal_part| × 10^32 + |register|)`
> 不消耗 register, 保持审计完整性。

#### 审计方法

| 方法 | 说明 |
|---|---|
| `evidence_hash()` | SHA-256 证据哈希 (含 ops_log_fragments, 8A 方案) |
| `verify(expected_hash)` | 验证哈希 (检测篡改), 仅验证哈希, 不重建对象 |
| `to_canonical_json()` | canonical JSON 序列化 (确定性, 含 ops_log_fragments) |
| `_flush_ops_log(out_dir)` | 手动 flush ops_log 到外部文件 (半公开, 审计专用) |

> **反序列化约束**: 完整重建 QinNumber 对象并继续运算**必须用到秦数学引擎本身**, 第三方仅能验哈希、不能脱离秦数学重建内部状态 (商业护城河)。

> **deviation 不自动生成 (硬性规定)**: 引擎不提供 `compute_deviation()` 方法。deviation = R1 结果 对比 R0 结果, 由用户/审计员在外部跑两遍比对, 不以 R0 校正 R1。

#### 属性

| 属性 | 说明 |
|---|---|
| `value` | 浮点近似值 (仅显示, 不用于计算) |
| `to_decimal()` | 精确 Decimal 值 |
| `has_rounding` | 是否舍入过 (register≠0) |
| `register` | register 值 (乘法累积/除法最后余数, 累积阶段非负) |
| `_ops_log_fragments` | 已 flush 的外部文件路径列表 (8A 方案, 纳入 evidence_hash) |

#### 属性保护 (`__setattr__` 拦截)

| 属性 | 可否变 | 原因 |
|---|---|---|
| `truth_mode` (R0) | ❌ 不可变 | 全局真值基准, 链式运算中不可变 (运行时拦截抛 AttributeError) |
| `precision_k` | ❌ 不可变 | K 值变化后 `_scale` 变, 旧 register 失义 |
| `rounding` (R1) | ✅ 可变 | 每次运算可选, 设计本意 |
| `finalized` 前其余字段 | ✅ 可变 | — |
| `finalized` 后审计字段 (`integer_part/decimal_part/register/has_rounding/finalized`) | ❌ 不可变 | 已封存到证据链 |

#### 便捷函数

| 函数 | 说明 |
|---|---|
| `qin_sum(values)` | 链式求和 v1+v2+...+vn (不产生 sub-ULP, register 保留) |
| `qin_chain_multiply(values, rounding=...)` | 链式乘法 v1×v2×...×vn |
| `qin_chain_divide(values, rounding=...)` | 链式除法 v1÷v2÷...÷vn |

#### 异常

| 场景 | 异常类型 | 行为 |
|---|---|---|
| 除以零 | `ZeroDivisionError` | `a.divide(QinNumber(0,0))` 抛出 |
| `sqrt(负数)` | `ValueError` | `QinNumber(-1, 0).sqrt()` 抛出 |
| K 值不同比较 | `ValueError` | `a == b` (K 不同) 抛出 |
| R0 (truth_mode) 修改 | `AttributeError` | `x.truth_mode = ...` 抛出 |
| K (precision_k) 修改 | `AttributeError` | `x.precision_k = ...` 抛出 |
| finalized 后改审计字段 | `AttributeError` | `c.decimal_part = ...` 抛出 |

> **前置约束**: K=0、非法 rounding 在秦数学体系中不存在 (构造时即被 dataclass + `__setattr__` 保护拦截), 不需要运行时检测。

---

## 6. 舍入模式使用指南

### bit-identical 边界声明

> 秦数学承诺的跨平台 bit-identical 仅在**以下所有条件同时满足**时成立:
>
> 1. **同操作数顺序**: `a×b` 与 `b×a` 的 register 累积路径不同 (当前实现偶然对称, 但不保证);
> 2. **同运算符优先级**: `(a×b)×c` 与 `a×(b×c)` 的 register 累积路径不同, register 不保证一致, finalize 后 K 位账面**通常**一致但不保证极端边界;
> 3. **同 R1 模式链**: 链中每步 `rounding` 参数完全一致;
> 4. **同 K 值**: 比较或复现必须使用相同 `precision_k`;
> 5. **同秦数学引擎版本**: `evidence_hash` 随版本升级可能不兼容。
>
> 不保证交换律、结合律在 register 层面成立; 这是 sub-ULP 累积路径不同的固有结果, 不影响 K 位账面精度 (finalize 后通常一致)。

### 模式选择表

| 模式 | 适用场景 | 链式乘法 | 链式除法 | 加减法 | 备注 |
|---|---|---|---|---|---|
| `HALF_EVEN` | 统计/金融 (默认) | ✓ | ✓ | ✓ (不触发舍入) | 推荐默认, 无偏舍入 |
| `HALF_UP` | 通用计算 | ✓ | ✓ | ✓ (不触发舍入) | 四舍五入 |
| `DOWN` | 金融保守计算 | ✓ | ✓ | ✓ (不触发舍入) | 向负无穷, 不高估 |
| `TOWARD_ZERO` | 物理截断 | ✓ | ✓ | ✓ (不触发舍入) | 快速近似, 向零截断 |
| `UP` | 上限估计 | ✓ | ✓ | ✓ (不触发舍入) | 向正无穷, 保守偏大 |
| `AWAY_FROM_ZERO` | 材料/核能 (安全余量) | ✓ | ✓ | ✓ (不触发舍入) | 远离零, 安全余量 |

> **注**: REGISTER_CARRY 不是舍入模式, 乘法默认总是开 (累积+满量进位)。
> R1 舍入模式只决定 finalize 时 K 边界怎么舍入。
> 加减法不产生 sub-ULP, 不触发 K 边界舍入。

### REGISTER_CARRY 乘法累积机制

REGISTER_CARRY 是乘法 32 进制寄存累积机制, **不是舍入模式**:
- 乘法运算中, sub-ULP 累积到 register
- register 满 10^32 时, 进位恢复到 decimal_part
- 这是乘法的固有行为, 不需要用户选择, 乘法默认总是开
- 与 TOWARD_ZERO 的本质区别: TOWARD_ZERO 截断后余数丢弃不进位; REGISTER_CARRY 截断后余数累积并最终进位, 恢复精度

> **在规则内恢复精度**: REGISTER_CARRY 的精度恢复是有边界条件的——
> - 纯乘法链 a×b×c: register 累积, 满 10^32 进位恢复 (在规则内恢复)
> - 混合运算 a×b÷c: 乘法累积满量部分已进位恢复; 未满量 sub-ULP 被除法覆盖, 不恢复
> - 除法链 a÷b÷c: 除法只存最后余数, 不累积不恢复

### 下溢场景 (定点算术固有特性)

当运算结果小于 `10^-K` 时, 结果会下溢到 0 或保留 1 ULP。

```python
# K=18 下, 0.5e-18 × 0.5e-18 = 2.5e-35 (超出 K=18 表示范围)
a = QinNumber(0, 5)  # 5e-18
b = QinNumber(0, 5)
r = a.multiply(b).finalize()
print(r)  # !0.000000000000000000 (下溢到 0)

# 解决方案: 提高精度 K
a = QinNumber(0, 5, precision_k=50)
b = QinNumber(0, 5, precision_k=50)
r = a.multiply(b).finalize()
print(r)  # !0.000...025 (正常)
```

此限制与 IEEE 754 浮点数的 subnormal 下溢一致, 是定点算术的固有特性, 非算法缺陷。

---

## 7. 已知限制

| 限制 | 说明 | 应对方法 |
|---|---|---|
| 下溢 | 结果 < 10⁻ᴷ 时归零 | 提高精度 K |
| **sub-sub-ULP 截断** | `_sub_ulp_to_register` 整数除法截断 10⁻³² 量级以下的 sub-sub-ULP (固有限制, 无限延伸不可行) | 32 位 register 已足够 sub-ULP 精度, 损失极小 |
| **K 值不同比较抛异常** | K 值不同的 QinNumber 比较会抛 ValueError (维度不同) | 比较前确保 K 一致 |
| 固定 K 位精度 | 非任意精度 | 根据场景选择 K |

> **sub-sub-ULP 截断详解**:
> - sub-ULP 映射: `register_val = sub_ulp × 10^32 // scale` (整数除法)
> - 整数除法会截断小数部分, 即"sub-sub-ULP" (超过 32 位精度的比例分数)
> - 例: sub_ulp=1, scale=3 → 10^32 // 3 = 33...3(32 个 3), 截断余数=1
> - 这是定点算术的固有限制, 如同 IEEE 754 的 subnormal 下溢, 无法避免 (无限延伸三级、四级 register 不可行)
> - 损失量级: 10⁻³² 以下的比例分数, 在 K 位精度下完全可以接受

> **满量进位无损的边界**: "满量进位无损"指**在二级 32 位范围内无损** (整数除法+取模, 信息不丢失)。超过二级 32 位外的 sub-sub-ULP 会被截断, 这是定点算术固有特性, 不能宣称"绝对无损"。

---

## 8. 适用场景

### 适合的场景

| 场景 | 原因 | 推荐 K | 推荐 R0 |
|---|---|---|---|
| 金融结算 | 需要 bit-identical + 审计 + 余额加减 + 阈值比较 | 18 或 8 (加密货币) | HALF_EVEN |
| 医学影像重建 | 跨设备 bit-identical + 阈值比较 | 18 | HALF_EVEN |
| 物理模拟 | 确定性 + 可复现 | 30-32 | TOWARD_ZERO |
| 太阳耀斑预测 | 安全余量 + 审计 | 30 | AWAY_FROM_ZERO |
| 区块链智能合约 | 链上确定性 + 证据链 | 18 | HALF_EVEN |
| 大整数资产管理 | 无界整数 + 高精度小数 | 18 | HALF_EVEN |

### 不适合的场景

| 场景 | 原因 | 替代方案 |
|---|---|---|
| 游戏物理 | 速度优先 | IEEE 754 浮点 |
| 实时渲染 | 速度优先 | IEEE 754 浮点 |
| 符号计算 | 需任意精度动态扩展 | Mathematica |

---

## 9. 附录: 设计哲学

### 多级整数结构的本质

秦数学本质是**多级整数运算**, 不是简单的 K 位定点:
- 第0级整数无界, 支持 10^77+ 不溢出
- 第1级小数有界 K 位, 是用户可见的精度
- 第2级 register 暗藏 k=32 位, 是 sub-ULP 的影子
- 级间满量进位: register 满 10^32 → decimal_part; decimal_part 满 10^K → integer_part
- 这跟普通的定点计算差太远太远了

### 真值的定义

真值 = 行业常规标准。**不是脱离秦数学体系的"理想真值"**, 而是 R0 模式下 K+32 位精度内的工程约定。
R0 走同一套秦数学体系 (多级整数、register、REGISTER_CARRY 等), 与 R1 在体系内比对, 不校正 R1。

> **R0 与 R1 的关系**:
> - **秦数学 R0 模式** = 用 R0 当作 R1 跑一遍运算 (R0 既是 truth_mode 也是 rounding)
> - **秦数学 R1 模式** = 用 R1 跑一遍运算 (R0 是 truth_mode, R1 是 rounding)
> - deviation = R1 结果 对比 R0 结果 (两者都是秦数学跑出来的, 用户外部比对)
> - **不以 R0 标准自动校正 R1** (硬性规定, 引擎不提供自动 deviation 计算)

### register vs deviation (命名层级)

| 层级 | 名称 | 时机 | 性质 |
|---|---|---|---|
| 单步运算 | register (寄存) | 运算过程中 | 中性记录, 不修正 |
| 乘法基础规则 | REGISTER_CARRY 满量进位 | 乘法运算中 | 机制2(级间进位)的延伸, 非舍入修正 |
| finalize 阶段 | K 边界舍入 | 最后一步 | 按 R1 舍入, register 影子决定, **是正常计算的最后一步, 不是修正** |
| 用户外部比对 | deviation (偏差) | 审计阶段 | R1 结果 对比 R0 结果, 用户外部跑两遍得出, 仅审计, 引擎不自动生成 |

### 为什么不叫 correction (修正)

- `correction` 暗示"要改正", 容易误以为要修改结果
- `register` / `deviation` 是中性词, 只记录偏离多少, 不暗示修正
- **finalize 是正常计算的最后一步 K 边界舍入, 不是 correction (修正)**
- "不修正"指**运算中** (multiply/divide/sqrt/add/sub) 不做 K 边界舍入; finalize 做 K 边界舍入是计算本身的最后一步, **不是对之前结果的修正**

### R0 真值不是"圣经"

- R0 只定义"真值基准"供最终审计比对
- 用户的 R1 选择才是决定计算结果的模式
- 无前缀 (`!` 缺失) 只表示"这步没舍入", 不代表"等于 R0 真值"
- 如果所有结果都必须等于 R0 真值, 用户选不同 R1 就毫无意义

### "!" 标识的意义

- 无前缀: 未舍入 (register=0), 无 sub-ULP
- `!` 前缀: 已舍入 (register≠0), 有 register 记录

**注意**: `!` 只表示"这步发生了舍入", 不表示"偏离了真值"。

### 为什么 REGISTER_CARRY 不是舍入模式

- REGISTER_CARRY 是乘法的**固有累积行为**, 不是用户可选的舍入模式
- 乘法默认总是累积 sub-ULP 到 register, register 满量进位恢复
- R1 舍入模式 (HALF_UP/HALF_EVEN 等) 只决定最后一步 finalize 时 K 边界怎么舍入
- 两者职责不同: REGISTER_CARRY 管累积进位, R1 管边界舍入
- 将 REGISTER_CARRY 作为舍入模式会导致语义混淆 (它不决定 K 边界怎么舍入)

### 5A 方案: 负 register 满量进位规则

秦数学的乘法采用**绝对值运算**, 所以累积阶段 register 始终非负。数字符号由 `integer_part` / `decimal_part` 承载, register 是绝对值的 sub-ULP 影子。

满量进位规则仅处理非负 register: `register ≥ 10^32` 时进位恢复到 `decimal_part`。

负 register 只在 finalize 后出现 (表示超计, 如 AWAY_FROM_ZERO 模式), 仅审计用, 不参与累积——这样避免乘法累积时符号混乱, 与"除法不累积避免符号反转"的设计哲学一致。

###  详细哈希

`_ops_log_fragments` (已 flush 的外部文件路径列表) 纳入 `evidence_hash` 计算:
- 任何外部审计文件被替换/丢失/重命名, evidence_hash 都会变化
- 保证跨文档审计链完整性
- 单 register 被覆盖也不影响审计完整性——哈希绑定全部 ops_log + fragments, 审计员可从 ops_log 重建历史 sub-ULP 状态, **审计完整性靠哈希绑定 ops_log, 不靠 register 完整保留**

### finalize 后重启新链

- finalized=True 后调用 `multiply / divide / sqrt / add / sub` 时:
  1. 隐式 `copy()` 一份副本
  2. 旧 `register` **清零抛弃** (影子已在 finalize 消耗, 用于 K 边界舍入 + 审计)
  3. `finalized=False`、`ops_count=0`、`_ops_log=[]` (新链开始)
  4. 按"当前账面 K 位"为新基线开始新一轮运算

---


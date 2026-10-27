This open-source project is available for users to download and use at their own discretion. Our team assumes no direct, indirect or derivative liability or disputes arising from the use of the code in this project.
--------------------------------------------------------------------------------------------------------------------------------------

# QinMath 

## What It Does

- **Exact arithmetic**: addition, subtraction, multiplication exact within configured precision; division and square root bounded to a deterministic grid.
- **Six explicit rounding modes**: DOWN, UP, TOWARD_ZERO, AWAY_FROM_ZERO, HALF_UP, HALF_EVEN.
- **Dual-mode system**: immutable truth mode (R0) for audit baseline; per-operation rounding mode (R1) for computation.
- **Unbounded integers**: no overflow at 10^77 and beyond.
- **Tamper-evident audit trail**: canonical JSON + SHA-256 evidence hash; operation logs auto-flush to disk.
- **Bit-identical results**: same sequence, same rounding mode, same precision → identical output across platforms.
- **Property protection**: frozen configuration fields; post-finalize immutability.

纯 Python 高精度算术引擎。加、减、乘、除、开方，全程微小精度丢失。


## 安装

```bash
pip install .
```

Python 3.10+。无需编译，零三方依赖。

---

## 五分钟上手

```python
from qinmath_engine import QinNumber, RoundingMode

# 任意精度字符串 → QinNumber
a = QinNumber.from_string("3.14159265358979323846", k=20)
b = QinNumber.from_string("2.71828182845904523536", k=20)

# 链式运算 — 中间步骤精度不丢
product = a.multiply(b, RoundingMode.HALF_EVEN)
ratio   = a.divide(b, RoundingMode.TOWARD_ZERO)
root    = a.sqrt(RoundingMode.HALF_UP)

# finalize: 按你挑的舍入模式一次性收尾
print(product.finalize())   # 8.53973422267356706545
print(ratio.finalize())     # 1.15572734979092171791

# 构造方式
QinNumber.from_int(42, k=18)
QinNumber.from_float(3.14, k=20)
QinNumber.from_decimal(Decimal("1.61803398874989484820"))
```

### 六种舍入模式

```
RoundingMode.DOWN          向负无穷
RoundingMode.UP            向正无穷
RoundingMode.TOWARD_ZERO  向零截断
RoundingMode.AWAY_FROM_ZERO 远离零
RoundingMode.HALF_UP       四舍五入
RoundingMode.HALF_EVEN     银行家舍入
```

### 比较、取证

```python
a == b     a < b     a <= b     a >= b     a > b     hash(a)
a.evidence_hash()   # SHA-256, 每次运算后可审计
```

---

## 项目文件

```
qinmath-engine/
├── LICENSE
├── CLA.md              个人贡献者许可协议
├── CLA-ENTITY.md       实体贡献者许可协议
├── CHANGELOG.md
├── MANIFEST.in
├── pyproject.toml
├── README.md
└── qinmath_engine/
    ├── __init__.py
    ├── _core.py
    └── py.typed
```

---

## 兼容性

| Python | 状态 |
|--------|------|
| 3.10   | ✅   |
| 3.11   | ✅   |
| 3.12   | ✅   |
| 3.13   | ✅   |

纯 Python，跨平台，无 C 扩展，无平台 wheel 依赖。

---

## 许可证 & 贡献

**GNU Affero General Public License v3.0 only (AGPL-3.0-only)**。
修改版/网络服务部署后必须向用户公开源码。专有许可请联系 QinMath Team。

贡献前请先签署 CLA：

- 个人 → [CLA.md](./CLA.md)
- 公司 / 组织 → [CLA-ENTITY.md](./CLA-ENTITY.md)

---

## 引用

> QinMath, *QinMath Engine*, v2.0.0, 2026.

## 联系方式 contact

qintech2026@163.com

## 打赏

if it helps please donate under the name QinMath to the organization that you consider meaningful, which is affiliated with the United Nations.

## 作者 Author
Qin with the help of Chen and AI.



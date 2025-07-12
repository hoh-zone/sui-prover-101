## CetusProtocol/integer-mate 库介绍
integer-mate 是 Cetus Protocol 开发的一个 Move 数学库，专注于 高效、安全的定点数（Fixed-Point）和整数运算，
主要用于Sui DeFi 场景（如集中流动性做市商 CLMM 中的价格计算、交易手续费计算等）。

## 核心功能
1. 定点数（Fixed-Point）运算
- 提供 高精度的十进制计算，避免浮点数在区块链环境中的不稳定性。
- 适用于 价格区间、手续费、流动性份额 等 DeFi 关键计算。

2. 安全的整数运算
- 防止溢出（Overflow）和下溢（Underflow），确保智能合约的安全性。
- 支持 加减乘除、指数、开方 等常见数学操作。

3. 与 Cetus 协议深度集成
- 用于 集中流动性做市（CLMM） 的价格计算、滑点控制等核心逻辑。

### 使用场景
- DEX 价格计算（如 Uniswap v3 风格的 CLMM）。
- 流动性池份额管理（计算 LP 代币的价值）。
- 交易手续费计算（精确到小数点后多位）。
- 任何需要高精度整数运算的 DeFi 或区块链应用。

## 如何集成？
```toml
[dependencies]
integer-mate = { git = "https://github.com/CetusProtocol/integer-mate" }
```


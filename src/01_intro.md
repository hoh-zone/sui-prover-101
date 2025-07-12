## Sui Prover 是什么
Sui Prover 是一种专为 Sui 区块链设计的智能合约语言，它在 Move 语言的基础上进行了扩展，强调安全性、可组合性和形式化验证。形式化证明在 Sui Move 中的应用主要是通过数学方法严格验证合约逻辑，确保其符合预期行为，避免漏洞。
## 什么是形式化证明


## 为什么 Sui Move 适合形式化证明？
   Sui Move 在设计时就考虑了形式化验证，具有以下特点：
- 基于线性类型（Linear Types）：资源（如代币）不能被复制或丢弃，确保资产安全。
- 显式所有权（Explicit Ownership）：所有权的转移必须显式声明，减少权限漏洞。
- 模块化设计：代码结构清晰，便于形式化建模。
- 静态验证支持：Move Prover（形式化验证工具）可直接集成到开发流程中。

## 安装 
macos brew 安装
```shell
brew install asymptotic-code/sui-prover/sui-prover
```

cargo 安装
```shell
cargo 
```

本地编译

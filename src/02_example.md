更新 `Move.toml` 文件以使用隐式依赖项
Sui 验证器遵循了 Move.toml 文件中近期的惯例，即依赖隐式依赖项。
因此，您需要移除对 Sui 和 MoveStdlib 的任何直接依赖关系。
有关更多背景信息，请参阅此消息。
// 如果`move.toml`有这一行请删除:
```toml
Sui = { git = "https://github.com/MystenLabs/sui.git", subdir = "crates/sui-framework/packages/sui-framework", rev = "framework/testnet", override = true }
```

如果您确实需要直接引用 Sui，请将相关规范放在单独的文件中。
基本用法
您可以在此处找到本指南中使用的完整示例：https://github.com/asymptotic-code/sui-kit/tree/main/examples/guide
要使用 Sui 验证器，您需要为您的智能合约编写规范。
然后，Sui 验证器将尝试证明您的智能合约满足这些规范。
示例
让我们以一个简单的示例来说明一下，即一个简化版的 LP（流动性池）智能合约中的取款功能：
```move
module amm::simple_lp;

use sui::balance::{Balance, Supply, zero};

public struct LP<phantom T> has drop {}

public struct Pool<phantom T> has store {
    balance: Balance<T>,
    shares: Supply<LP<T>>,
}

public fun withdraw<T>(pool: &mut Pool<T>, shares_in: Balance<LP<T>>): Balance<T> {
    if (shares_in.value() == 0) {
        shares_in.destroy_zero();
        return zero()
    };

    let balance = pool.balance.value();
    let shares = pool.shares.supply_value();

    let balance_to_withdraw = (((shares_in.value() as u128) * (balance as u128)) / (shares as u128)) as u64;

    pool.shares.decrease_supply(shares_in);
    pool.balance.split(balance_to_withdraw)
}
```
为了验证取款功能是否正确，我们需要为其编写一份规范说明。该规范应描述该功能的预期行为。
规范的结构
规范通常具有以下结构：
```move
#[spec(prove)]
fun withdraw_spec<T>(pool: &mut Pool<T>, shares_in: Balance<LP<T>>): Balance<T> {

    // 函数的参数所假定满足的条件
    let result = withdraw(pool, shares_in);

    // 作为函数调用的结果所必须满足的条件
    result
}
```

让我们来详细分析上述内容：
该规范将定义一个新的函数`withdraw_spec`，它与`withdraw`函数具有相同的参数，并返回相同的类型。
该函数通过 #[spec(prove)] 进行注解，以表明它是规范，并且将使用 Sui 验证器进行验证。
规范函数的主体通常包括：
首先，指定在函数参数上所假定成立的条件
然后调用要验证的函数
接着指定函数调用所产生的条件必须成立
最后返回函数调用的结果
示例规范
让我们编写一个规范，说明在提取资金时股票的价格不应下降：
```sui move
#[spec(prove)]
fun withdraw_spec<T>(pool: &mut Pool<T>, shares_in: Balance<LP<T>>): Balance<T> {
    
    requires(shares_in.value() <= pool.shares.supply_value());

    let old_pool = old!(pool);

    let result = withdraw(pool, shares_in);

    let old_balance = old_pool.balance.value().to_int();
    let new_balance = pool.balance.value().to_int();

    let old_shares = old_pool.shares.supply_value().to_int();
    let new_shares = pool.shares.supply_value().to_int();

    ensures(new_shares.mul(old_balance).lte(old_shares.mul(new_balance)));

    result
}
```
让我们来详细解读一下这个规范：
我们明确了函数参数所必须满足的条件。要提取的股份数量必须小于或等于池中的总股份数量：
```
requires(shares_in.value() <= pool.shares.supply_value());
```

`requires` 是一个`only-spec`的关键字，用于指定函数参数所应满足的条件。
我们将池的旧状态保存在`old_pool`中：
```move
let old_pool = old!(pool);
```

我们称要验证的函数为：
```move
let result = withdraw(pool, shares_in);
```
我们计算出旧余额和新余额以及各自的份额，并将它们转换为无界整数：
``` move
let old_balance = old_pool.balance.value().to_int();
let new_balance = pool.balance.value().to_int();

let old_shares = old_pool.shares.supply_value().to_int();
let new_shares = pool.shares.supply_value().to_int();
```
无界整数是一种`仅在规范中定义`的类型，它允许我们编写条件而无需担心溢出问题。这种类型仅在规范环境中可用。
我们规定了作为函数调用效果所必须满足的条件。当提取资金时，股票的价格不应下降：

```
ensures(new_shares.mul(old_balance).lte(old_shares.mul(new_balance)));
```

`ensures` 是一个仅用于规范的关键字，它指定了函数调用所产生的必须满足的条件。
`mul` 和 `lte` 是上述未受限制整数类型上的仅用于规范的运算符。
最后，我们返回结果。
运行 Sui 验证器
要运行 Sui 验证器，您需要先安装并配置 Sui 验证器。
从 move.toml 目录中运行以下命令即可运行 Sui 验证器：
```shell
sui-prover
```
使用`幽灵`变量
`幽灵`变量是一种用于声明仅在规范中使用的全局变量的方法。
它们特别适用于在不同规范之间传递信息。
以上述示例为例，接下来：
添加一个功能，使其在进行大额取款时触发事件，
然后添加一个幽灵变量来检查该事件是否正确触发。
更新后的示例
首先，让我们将事件触发功能添加到原始示例中：

```move
module amm::simple_lp;

use sui::balance::{Balance, Supply, zero};
use sui::event;

#[spec_only]
use prover::prover::{requires, ensures, asserts, old};
#[spec_only]
use prover::ghost::{declare_global, global};

public struct LP<phantom T> has drop {}

public struct Pool<phantom T> has store {
    balance: Balance<T>,
    shares: Supply<LP<T>>,
}

const LARGE_WITHDRAW_AMOUNT: u64 = 10000;

public struct LargeWithdrawEvent has copy, drop {}

fun emit_large_withdraw_event() {
    event::emit(LargeWithdrawEvent { });
}

public fun withdraw<T>(pool: &mut Pool<T>, shares_in: Balance<LP<T>>): Balance<T> {
    if (shares_in.value() == 0) {
        shares_in.destroy_zero();
        return zero()
    };

    let balance = pool.balance.value();
    let shares = pool.shares.supply_value();
    let shares_in_value = shares_in.value();

    let balance_to_withdraw = (((shares_in.value() as u128) * (balance as u128)) / (shares as u128)) as u64;

    pool.shares.decrease_supply(shares_in);

    if (shares_in_value >= LARGE_WITHDRAW_AMOUNT) {
        emit_large_withdraw_event();
    };

    pool.balance.split(balance_to_withdraw)
}
```


我们进行了以下更改：
添加了事件模块，
声明了`LARGE_WITHDRAW_AMOUNT`限制条件，
添加了一个`LargeWithdrawEvent`结构体，
添加了用于发出事件的`event::emit`函数。
更新了规范
现在，为了检查该事件是否正确发出，我们需要声明一个虚拟变量，并检查在事件发出后该变量是否被设置为`真`：
首先，我们在规范的开头添加以下行来声明这个虚拟变量：
```move
declare_global<LargeWithdrawEvent, bool>();
```
然后，我们检查在事件发出后该值是否已被设置为`真`。
我们可以使用常规的 if 语句来实现这一点，而在规范中它会成为一个逻辑条件。
```move
if (shares_in_value >= LARGE_WITHDRAW_AMOUNT) {
    ensures(*global<LargeWithdrawEvent, bool>());
};
```

最后，我们确保原始函数在触发事件时需要先设置`幽灵`变量：
```move
requires(*global<LargeWithdrawEvent, bool>());
```


完整的代码如下所示。
完整的示例以供参考
以下是完整的示例供参考：

``` move
module amm::simple_lp;

use sui::balance::{Balance, Supply, zero};
use sui::event;

#[spec_only]
use prover::prover::{requires, ensures, asserts, old};
#[spec_only]
use prover::ghost::{declare_global, global};

public struct LP<phantom T> has drop {}

public struct Pool<phantom T> has store {
balance: Balance<T>,
shares: Supply<LP<T>>,
}

const LARGE_WITHDRAW_AMOUNT: u64 = 10000;

public struct LargeWithdrawEvent has copy, drop {}

fun emit_large_withdraw_event() {
event::emit(LargeWithdrawEvent { });
requires(*global<LargeWithdrawEvent, bool>());
}

public fun withdraw<T>(pool: &mut Pool<T>, shares_in: Balance<LP<T>>): Balance<T> {
if (shares_in.value() == 0) {
shares_in.destroy_zero();
return zero()
};

    let balance = pool.balance.value();
    let shares = pool.shares.supply_value();
    let shares_in_value = shares_in.value();

    let balance_to_withdraw = (((shares_in.value() as u128) * (balance as u128)) / (shares as u128)) as u64;

    pool.shares.decrease_supply(shares_in);

    if (shares_in_value >= LARGE_WITHDRAW_AMOUNT) {
        emit_large_withdraw_event();
    };

    pool.balance.split(balance_to_withdraw)
}

// Verify that the price of the token is not decreased by withdrawing liquidity
#[spec(prove)]
fun withdraw_spec<T>(pool: &mut Pool<T>, shares_in: Balance<LP<T>>): Balance<T> {
requires(shares_in.value() <= pool.shares.supply_value());

    declare_global<LargeWithdrawEvent, bool>();

    let old_pool = old!(pool);
    let shares_in_value = shares_in.value();

    let result = withdraw(pool, shares_in);

    let old_balance = old_pool.balance.value().to_int();
    let new_balance = pool.balance.value().to_int();

    let old_shares = old_pool.shares.supply_value().to_int();
    let new_shares = pool.shares.supply_value().to_int();

    ensures(new_shares.mul(old_balance).lte(old_shares.mul(new_balance)));

    if (shares_in_value >= LARGE_WITHDRAW_AMOUNT) {
        ensures(*global<LargeWithdrawEvent, bool>());
    };

    result
}

```
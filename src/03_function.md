##  `prover` 模块

`prover` 模块是一个仅包含规范内容的模块，它为编写规范提供了基础构建要素。
### `requires`函数

`requires`函数用于指定该函数参数所应满足的条件。

```rust
requires(shares_in.value() <= pool.shares.supply_value());
```

###  `ensures` 函数

`ensures`函数用于指定在调用该函数后必须满足的条件。
```rust
ensures(new_shares.mul(old_balance).lte(old_shares.mul(new_balance)));
```

### `asserts`函数

`asserts`函数用于指定必须始终满足的条件。
```rust
asserts(shares_in.value() <= pool.shares.supply_value());
```

###  `old` 宏函数

`old` 这个宏用于指代函数调用之前对象的状态。
```rust
let old_pool = old!(pool);
```

###  `to_int` 方法

`to_int`方法用于将固定精度的无符号整数转换为无限制范围的整数。
无限制范围的整数仅在执行规范时可用。

```rust
let x = 10u64.to_int();
```

###  `to_real` 方法
`to_real` 方法用于将固定精度的无符号整数转换为任意精度的实数。
这有助于确定舍入方向。
实数仅在执行规范时可用。

```rust
let x = 10u64.to_real();
```

### Ghost 变量

“幽灵变量”是指那些仅在规范中使用的全局变量。
这些变量是通过 `ghost::declare_global` 和 `ghost:：declare_global_mut` 函数来声明的：
```rust
ghost::declare_global_mut<Name, Type>();
```

该声明接受两个类型级别的参数：幽灵变量的名称及其实际类型。
在大多数情况下，幽灵变量的类型级别名称要么是用户定义的类型，要么只是新声明的仅用于规范的结构体。例如：

```rust
public struct MyGhostVariableName {}
```

然后，这个“幽灵”变量就可以在“要求”、“保证”和“断言”函数中使用了：例如：

```rust
requires(ghost::global<MyGhostVariableName, _>() == true);
```

### Loop `invariant`

循环不变量是指在循环的所有迭代过程中都必须满足的条件。
如果规格说明中包含了在循环内部会被修改的变量的相关条件，那么就必须指定循环不变量。
该不变量是通过在循环之前调用`invariant`宏来指定的。

```rust
invariant!(|| {
    ensures(i <= n);
});
while (i < n) {
    i = i + 1;
}
```

### Object `invariant`

对象不变性是指对于某一特定类型的所有对象而言，都必须满足的条件。
通过创建一个名为 `<type_name>_inv` 的新函数，并在该函数上添加 `#[spec_only]` 标记，即可声明一个对象不变量。

```rust
#[spec_only]
public fun MyType_inv(self: &MyType): bool {
    ...
}
```

该函数必须接受一个参数，该参数是一个指向对象的引用，并返回一个布尔值，该值表示对象的不变性是否成立。
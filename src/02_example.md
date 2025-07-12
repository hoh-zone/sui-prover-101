
Update Move.toml to use implicit dependencies
The Sui Prover relies on the recent convention for Move.toml to rely on implicit dependencies. So you need to remove any direct dependencies to Sui and MoveStdlib . See this message for more context.
// delete this line:
```toml
Sui = { git = "https://github.com/MystenLabs/sui.git", subdir = "crates/sui-framework/packages/sui-framework", rev = "framework/testnet", override = true }
```

If you do need to reference Sui directly, put the specs in a separate package.
Basic usage
You can find the complete example used in this guide here: https://github.com/asymptotic-code/sui-kit/tree/main/examples/guide
To use the Sui Prover you need to write specifications for your smart contract. The Sui Prover will then attempt to prove that your smart contract satisfies these specifications.
Example
Let's consider a simple example, the withdraw function of a simplified LP (Liquidity Pool) smart contract:
```sui move

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
​
To verify that the withdraw function is correct, we need to write a specification for it. The specification should describe the expected behavior of the function.
The structure of a specification
Specs usually have the following structure:
#[spec(prove)]
fun withdraw_spec<T>(pool: &mut Pool<T>, shares_in: Balance<LP<T>>): Balance<T> {

    // Conditions which are assumed to hold on the arguments of the function

    let result = withdraw(pool, shares_in);

    // Conditions which must hold as an effect of the function call

    result
}
```

Let's break down the above:
The specification will be a new function withdraw_spec that takes the same arguments as the withdraw function and returns the same type.
The function is annotated with #[spec(prove)] to indicate that it is a specification and that it will be verified using the Sui Prover.
The body of the spec function usually:
specifies the conditions which are assumed to hold on the arguments of the function
then calls the function to be verified
specifies the conditions which must hold as an effect of the function call
returns the result of the function call
Example specification
Let's write a specification that says that the price of a share should not decrease when withdrawing funds:
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
Let's break down the specification:
We specify the conditions which must hold on the arguments of the function. The number of shares to withdraw must be less than or equal to the total number of shares in the pool:
```
requires(shares_in.value() <= pool.shares.supply_value());
```

requires is a spec-only keyword that specifies a condition which is assumed to hold on the arguments of the function.
We save the old state of the pool in old_pool:
```move
let old_pool = old!(pool);
```

We call the function to be verified:
```move
let result = withdraw(pool, shares_in);
```

We compute the old and new balances and shares, and convert them to unbounded integers:
```
let old_balance = old_pool.balance.value().to_int();
let new_balance = pool.balance.value().to_int();

let old_shares = old_pool.shares.supply_value().to_int();
let new_shares = pool.shares.supply_value().to_int();
```
Unbounded integers are a spec-only type which allow writing conditions without having to worry about overflows. The are only available in the spec environment.
We specify the conditions which must hold as an effect of the function call. The price of a share should not decrease when withdrawing funds:

```
ensures(new_shares.mul(old_balance).lte(old_shares.mul(new_balance)));
```

ensures is a spec-only keyword that specifies a condition which must hold as an effect of the function call.
mul and lte are spec-only operators over the unbounded integer types described above.
Finally, we return result.
Running the Sui Prover
To run the Sui Prover, you need to have the Sui Prover installed and configured.
Run the following command from the move.toml directory to run the Sui Prover:
```move
sui-prover
```
Using ghost variables
Ghost variables are a way to declare global variables that are only used in specifications.
They are particularly useful for propagating information between different specifications.
Starting with the example above, let's:
add functionality to that emits an event on a large withdrawal,
then add a ghost variable to check that the event is correctly emitted.
Updated example
First, let's add the event emission to the original example:

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


We made the following changes:
added the event module,
declared the LARGE_WITHDRAW_AMOUNT bound,
added a LargeWithdrawEvent struct,
added the event::emit function to emit the event.
Updated specification
Now, to check that the event is correctly emitted, we need to declare a ghost variable and check that it is set to true after the event is emitted:
First, we declare the ghost variable by adding the following line to the beginning of the specification:
```sui move
declare_global<LargeWithdrawEvent, bool>();
```
Then, we check that it is set to true after the event is emitted.
We can do this using a regular if statement, which becomes a logical condition in the specification.
```sui move
if (shares_in_value >= LARGE_WITHDRAW_AMOUNT) {
ensures(*global<LargeWithdrawEvent, bool>());
};
```

Finally, we make sure the original function requires the ghost variable to be set when the event is emitted:
```sui move
requires(*global<LargeWithdrawEvent, bool>());
```


The full code is available below.
Full example for reference
And here is the full example for reference:

```sui move


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
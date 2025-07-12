

[prover](https://github.com/asymptotic-code/sui-prover/blob/main/packages/prover/sources/prover.move)
```move
module prover::prover;

#[spec_only]
native public fun requires(p: bool);
#[spec_only]
native public fun ensures(p: bool);
#[spec_only]
native public fun asserts(p: bool);
#[spec_only]
public macro fun invariant($invariants: ||) {
    invariant_begin();
    $invariants();
    invariant_end();
}

public fun implies(p: bool, q: bool): bool {
    !p || q
}

#[spec_only]
native public fun invariant_begin();
#[spec_only]
native public fun invariant_end();

#[spec_only]
native public fun val<T>(x: &T): T;
#[spec]
fun val_spec<T>(x: &T): T {
    let result = val(x);

    ensures(result == x);

    result
}

#[spec_only]
native public fun ref<T>(x: T): &T;
#[spec]
fun ref_spec<T>(x: T): &T {
    let old_x = val(&x);

    let result = ref(x);

    ensures(result == old_x);
    drop(old_x);

    result
}

#[spec_only]
native public fun drop<T>(x: T);
#[spec]
fun drop_spec<T>(x: T) {
    drop(x);
}

#[spec_only]
public macro fun old<$T>($x: &$T): &$T {
    ref(val($x))
}

#[spec_only]
native public fun fresh<T>(): T;
#[spec]
fun fresh_spec<T>(): T {
    fresh()
}

#[spec_only]
#[allow(unused)]
native fun type_inv<T>(x: &T): bool;
```


[ghost](https://github.com/asymptotic-code/sui-prover/blob/main/packages/prover/sources/ghost.move)
```sui move
module prover::ghost;

#[spec_only]
use prover::prover;

#[spec_only]
public native fun global<T, U>(): &U;

#[spec_only]
public native fun set<T, U>(x: &U);

#[spec]
public fun set_spec<T, U>(x: &U) {
  declare_global_mut<T, U>();
  set<T, U>(x);
  prover::ensures(global<T, U>() == x);
}

#[spec_only]
public native fun borrow_mut<T, U>(): &mut U;

#[spec_only]
public native fun declare_global<T, U>();
#[spec_only]
public native fun declare_global_mut<T, U>();

#[spec_only]
#[allow(unused)]
native fun havoc_global<T, U>();
```

[log](https://github.com/asymptotic-code/sui-prover/blob/main/packages/prover/sources/log.move)
```sui move
module prover::log;

#[spec_only]
public native fun text(x: vector<u8>);

#[spec_only]
public native fun var<T>(x: &T);

#[spec_only]
public native fun ghost<T, U>();
```
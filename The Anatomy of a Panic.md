# **The Anatomy of a _panic_**

## Introduction

In **Rust**, there exists a family of functions known as _panics_ and _panic handlers_, which serve as an escape hatch whenever unrecoverable errors occur (or, if one is an incautious programmer, on every error). This article explores the mechanics of _panics_ in Rust, the never type (`!`), and how custom _panic handlers_ could enable more flexible control flow. We will examine how _panics_ work, how the never type is produced and used, and discuss the potential for alternative _panic handling_ strategies in Rust.

## The Nature of Panics

The mechanism by which _panics_ operate is not magical—the function signature of a _panic handler_ reveals all that is necessary to understand their behavior:
```rust
fn panic(panic_info: core::panic::PanicInfo) -> ! {
    // implementation details
}
```

Let us set aside the `panic_info` parameter for the moment, as the specific implementation of _panic_ is not our primary concern. The standard library provides a `panic` function that logs relevant information to standard error and terminates the program. However, by their nature, _panics_ should not be restricted to such behavior alone.

_Panics_ are, in essence, a control flow operator that communicates to the callsite: "I have nothing to return, and I will not return control to you." In Zig, this is referred to as `noreturn`, and in Rust, as `!`. This indicates that whenever a function with this signature is invoked, the compiler is informed that the callsite receives a value of type `!`, which is uninhabited and therefore unreachable. Such values may be produced in several ways:

-   code following a `return` statement
-   code following a `break`
-   an infinite loop that never terminates, etc.

## Producing the Never Type (`!`)

Let us, for a moment, treat `!` as a normal, instantiable type that can be passed around. If one attempts to implement a _panic_ function, it is reasonable to seek a means of producing a value of type `!`. Consider the following four illustrative examples:
```rust
fn panic(_: _) -> ! {
    // the return value of an infinite loop is `!`
    let value: ! = loop {};
    return value;
}
fn panic(_: _) -> ! {
    // had `return` not returned this function, the next `return` would work
    // as a valid panic handler. since the first `return` statement terminates
    // the function, the rest of the function is unreachable, and therefore a
    // `!` can be conjured from the result of a `return` statement.
    //
    // this is an invalid but illustrative example.
    let value: ! = return 5;
    return value;
}
fn panic(_: _) -> ! {
    // if this function had access to a labeled block, and the ability to
    // `break` to it (currently invalid in Rust), this block would suffice.
    let value: ! = break 'block None;
    return value;
}
fn panic(_: _) -> ! {
    // the `libc::exit` function terminates the program and causes the operating
    // system to reclaim the program's resources, effectively never
    // returning to the callsite. this is a method of communication between
    // abstraction levels. (OS <-> program)
    let value: ! = unsafe { libc::exit(1) };
    return value;
}
```

It is evident from these examples that conjuring and managing values of type `!` is not particularly difficult (even though such values never exist at runtime). Once again, a value of type `!` is never reachable at runtime.

## Coercion and Control Flow

Because a value of type `!` is never reachable at runtime, the compiler permits implicit coercion to **any other type**, as violating type state assumptions is impossible if the code is never executed:
```rust
let my_value: u32 = std::panic!(); // this panics, and the `my_value` variable and subsequent code
                                   // are never actually executed.
println!("Fun value: {}", my_value);
```

## Custom Panic Handlers and Block Control

From these deductions, one may conceive of novel _panic_ types that could be implemented in the standard library (or elsewhere). As observed in the previous examples, the second and third demonstrations are not valid in current Rust. We shall disregard the second case for now, as it remains unexplored, and the fourth, as it is somewhat more convoluted. Let us instead examine the third case.

If it were possible to define a function (or, more appropriately, a closure, as it would capture the surrounding environment) that could capture a block label from any parent scope—and if Rust permitted such usage—one could instantiate a _panic handler_ of the third type:
```rust
let x: Option<i32> = 'block: { ... };
match x {
    Some(val) => println!("Successful: {}", val),
    None => println!("'block panicked."),
}

// ... expanded:

let x: Option<i32> = 'block: {
    let panic_handler = || -> ! {
        break 'block None;
    };
    // ...
};
match x {
    Some(val) => println!("Successful: {}", val),
    None => println!("'block panicked."),
}

// ... expanded:

let x: Option<i32> = 'block: {
    let panic_handler = || -> ! {
        break 'block None;
    };
    let may_panic = foo(); // a function which may internally call panic!() or similar.
    Some(may_panic)
};
match x {
    Some(val) => println!("Successful: {}", val),
    None => println!("'block panicked."),
}
```

Within the `'block`, _panicking_ functions do not necessarily cause program termination. How is this possible? Let us examine the code from the perspective of the `'block` scope:
```rust
let panic_handler = ...;
let may_panic = foo(); // this function may panic, returning `!`, which can be coerced
                       // to the return type of `foo`. We do not care which specific
                       // panic function is invoked; we only need to know that control
                       // flow never returns here if `foo` panics.
```

It is that straightforward. Now, let us consider what this code _could_ compile down to in fully valid Rust:
```rust
let x: Option<i32> = 'block: {
    let may_panic = /* we must inline `foo` to reach `'block` */ {
        // assume this is the body of foo
        if some_external > 5 {
            // assume this is a call to `panic!()`
            break 'block None;
        } else {
            10
        }
    };
    Some(may_panic)
};
match x {
    Some(val) => println!("Successful: {}", val),
    None => println!("x panicked."),
}
```
In contemporary Rust, this code compiles and behaves as intended. There are no label hygiene issues, no lifetime complications—merely pure control flow. However, transitioning from this compiling code to a _panic-handler-agnostic_ version is the current challenge.

## Handler Selection and Inlining

The question arises: how does one determine which _panic handler_ the function `foo` should invoke if it internally calls `panic!()`? The answer is not immediately clear. Nevertheless, it should be feasible to determine the appropriate _panic handler_ at compile time via callsite analysis or perhaps the type system. As long as `foo` utilizes the `panic!()` macro, or invokes APIs compatible with `panic!()` such as `Option::unwrap`, we can control the destination of a _panicking_ `foo`. If `foo` internally employs mechanisms such as an infinite loop or a call to `libc::exit`, we cannot override those, as they are implementation details of `foo`, not the standard library or user code.

To understand why we need to inline `foo` to access the block label, consider how Rust's labeled blocks work: a `break` statement can only target labels that are visible in the current lexical scope. If `foo` is a separate function, it has no knowledge of or access to the label defined in its caller's scope. Therefore, if we want a _panic handler_ inside `foo` to break out of a block in the caller, the code that performs the `break` must be written directly within the block's scope. In this article, we currently leverage inlining to achieve this, but inlining is simply a detail to get compiling code, and not a requirement of this approach.

Consider a scenario in which `foo` may call `Option::unwrap`, which in turn calls `panic!()`:
```rust
fn foo(inside: Option<i32>) -> i32 {
    inside.map(|num| num + 5).unwrap()
}
let x: Option<i32> = 'block: {
    let panic_handler = || -> ! {
        break 'block None;
    };
    let may_panic = foo(None); // a function which may internally call panic!() or similar.
    Some(may_panic)
};
match x {
    Some(val) => println!("Successful: {}", val),
    None => println!("'block panicked."),
}
```

This may compile to the following:
```rust
fn foo(inside: Option<i32>) -> i32 {
    inside.unwrap()
}
let x: Option<i32> = 'block: {
    let may_panic = /* again, foo inlined to reach 'block */ {
        let optional  = None.map(|num| num + 5);
        // this is just `Option::unwrap`
        match optional {
            Some(t) => t,
            // call to `panic!()` replaced with our panic handler again.
            None => break 'block None,
        };
    };
    Some(may_panic)
};
match x {
    Some(val) => println!("Successful: {}", val),
    None => println!("'block panicked."),
}
```

Understanding handler selection and inlining is important for advanced Rust users who want to control program behavior in the face of _panics_, especially in low-level or systems programming contexts.

## Conclusion

In this article, I have explained that there may exist other forms of _panic handlers_ that do not require program termination, but instead utilize control flow to emulate _panicking_. This approach, which allows for optional _panic handlers_, grants low-level users greater flexibility in determining which parts of their programs may crash the running executable, and which are non-critical to program state. From the perspective of kernel development, this is essential if one wishes to avoid a failing slice index shutting down the entire system, without relying on both `panic = "unwind"` and explicit use of `catch_unwind` for every function call.

Thank you for reading. — Voxell Paladynee

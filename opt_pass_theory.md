# An abomination of a theoretical optimization pass

There is this optimization pass that has been lingering around in my mind for a long while, and I wanted to write it down somewhere. This is a theoretical optimization pass that I would love to tackle if I were a rustc maintainer, or contributor. Nonetheless, I am not, and I am not knowledgeable enough about the rust compiler in the first place. However, it would be really cool to see it in action in a eutopia.

## Where I got the idea from

While I was working on a project which revolved around the image crate, I was only using the entire image crate to convert a raw pixel buffer into png images. I was shocked when I saw the binary size was an entire 4 MB. Surely I can use the image crate as a lightweight PNG encoder, right?

...Right?

I decided I'd experiment with the binary size, so I wrote a really crappy image format conversion cli tool. Its sole purpose was to take a .png file as an argument, and spit out another image in a target format. It was really simple and short, fitting in only ~10 lines of code. Seriously, most of the lines were spent parsing and evaluating the arguments.

At first I tried baking the output format in, expecting to see a lot of dead code elimination (used by other image formats such as JPEG, BMP, TIFF etc.). I compiled the code with release mode and all of the optimizations I could think of, and noted the binary size. Then, I made it possible to choose the output image format (thus making nearly all enum variants for `ImageFormat` a valid input for the output function), and compiled the code again. I saw (almost) no change in binary size. How could that be? I was literally not using that many things in the baked in version. The baked-in format version should've been smaller! That's when I came across this idea:

# Automatic Const Generic Promotion / Function Signature Optimization / Function Argument Range Analysis

This theoretical optimization pass blurs the line between what is an implementation detail and what is behavior. The core of the optimization is to allow the compiler to prove when changing the signature of a function or changing the layout of a struct is sound to do under some conditions such as callsite range analysis and automatic const generic promotion.

The below snippets are functionally the same:

```rs
struct FactoryConst<const DEBUGMODE: bool>{};

impl<const DEBUGMODE: bool> FactoryConst<DEBUGMODE> {
    pub fn print_debug(&self) {
        if DEBUGMODE {
            println!("some log");
        }
    }
}

fn check_and_run_const<const COND: bool>() {
    if COND {
        let factory: FactoryConst<false> = FactoryConst {};
        some_operation(factory);
    }
}
```

```rs
struct Factory {
    debug_mode: bool,
};

impl Factory {
    pub fn print_debug(&self) {
        if self.debug_mode {
            println!("some log");
        }
    }
}

fn check_and_run_arg(cond: bool) {
    if cond {
        some_operation();
    }
}
```

However, `check_and_run_const` or `FactoryConst` can not be instantiated with a non-const value. This example does not work:

```rs
fn main() {
    let args = std::env::args().collect::<Vec<_>>();
    let cond: bool = args[1].parse().unwrap();
    check_and_run_const::<cond>();
}
```

Because `cond` is a value that is not known at compile time, it is impossible to invoke the const version of `check_and_run` with `cond` here. Therefore, it is much more natural and ergonomic to only have a `check_and_run_arg` function present in many libraries or functions, because a constant value can be demoted to a runtime value:

```rs
const COMPTIME_COND: bool = false;
fn main() {
    check_and_run_arg(COMPTIME_COND);
}
```

However, the opposite case is not true, you can not promote a compile-time known normal value (`let some_value = true`) to a `const` value if the function or the library does not provide you with a `FactoryConst` or `check_and_run_const` function explicitly. Sure, const propagation and const folding stuff are real and they work, but they don't actively support changing the function signature. This suggests we need a language-level support for automatic const generic promotion if we want to optimize the heck out of our programs and have the opportunity to eliminate much more dead code.

## Proposition

What I propose as a theoretical optimization pass is that we check for values passed in as an argument in a function signature, and depending on some conditions on the optimization pass, we can "promote" these arguments to behave as if they were a const generic:

```rs
fn main() {
    check_and_run_arg(false); // this gets changed to:
    /* check_and_run_arg_signature_changed::<false>(); */
}

fn check_and_run_arg_signature_changed<const COND: bool>() {
    if COND {
        let factory: FactoryConst<false> = Factory{};
        some_operation(factory);
    }
}
```

There are some nuances in the code block above. We must assume that the entire context of the program is the above, and we can do a "every location this function has been called" analysis (will be referred to as "callsite range analysis" from now on) for the function `check_and_run_arg`.

The details of the analysis is as follows:

1. We make sure the function is not exported or annotated with `#[no_mangle]`, or `extern` functions. Basically we avoid changing the function signature if the function is accessible from somewhere other than the compiler. In other words, the only valid way to instantiate and call the function has to go through the rust compiler. This way we avoid changing the function signatures for libraries and other stuff actively depending on the function signature.

-   We look for every callsite the function has been invoked, and apply these steps:
    -   For each argument, we follow these steps:
        -   We have a list of unobserved-variants for this specific argument of type **T** for this specific function **F**.
        -   We make sure the value with type **T** is not a runtime-known value. This check should be done after every other constant propagation passes, so that we can allow as many constants as possible to be promoted. If the value is instead a runtime-known value, this entire argument for the function **F** is ineligible for promotion.
        -   We are sure that the argument is a compile time known value, either via `const`s or `let`s or temporaries. We subtract the value from the range of possible values the argument type can be in. For example, a bool can be in 2 variants: `true` and `false`. Whenever an argument with a constant `false` is observed, we can remove `false` from the unobserved variants list.
-   We are left with a list of unobserved variants for functions eligible for const generic promotion. These variants, referenced anywhere in the code, are therefore unreachable in any way, shape or form, because we've made sure nobody can call the function again (outside of the compiler's view), and no calls can actively produce them. Moreover, the function is eligible for const generic promotion, because every time the function is called, a constant normal value is provided, and no runtime-known values are passed in to the function.

Therefore, the compiler is free to change the function signature by promoting the arguments into const generics and passing them to the functions as const generic arguments, and removes them from the argument section of the function. This way, less arguments are needed to be stored in the registers, and the compiler can do dead code analysis on the monomorphized functions and remove the unreachable code paths. Since the function signature is changed, the old function is no longer needed, and references to the function can be replaced with the new function signature and the arguments.

This however does come with some drawbacks:

-   If there is a single function being called with many different const-known values, the compiler will have to generate many different monomorphized functions for each const-known value. This can lead to a lot of code bloat, so a fundamental change on how to determine whether a function is a candidate for auto const generic promotion is needed.
-   This of course is a very destructive optimization and can break legacy or unsafe code relying on the function arguments and their order. Necessary precautions should be taken to make sure the optimization is sound and does not break any code.
-   This probably only helps cases where the function is being called only once. Because if you call it more than once with different const values, more than 1 function with a different signature is generated. Const generics are a wild tool, huh.

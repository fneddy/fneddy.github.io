# TOC
<!-- toc -->
# Rust arrays
In rust the Arrays size is part of the type. This means that an `[u8; 10]` and an `[u8; 32]` are completely incompatible.
In `C/C++` it is common practice to initialize an array with smaller data. 

```C
/* char array of*/
char DATA[32] = "Hello World";
```

Trying the same in rust:

```rust,compile_fail
pub static DATA: [u8; 32] = *b"Hello World";
```

```ignore
   |
5  | pub static Data: [u8; 32] = *b"Hello";
   |                             ^^^^^^^^^ expected an array with a size of 32, 
                                           found one with a size of 5
```

To my knowledge there is no way to to cast or automatically pad an array.
However since we got `const` functions in rust we have a solution for this.

# Rust const

`const` in rust is very similar to `constexpr` in `C++`. It is a completely different thing as `const` in `C`!!

From the rust documentation about `const`:
> Compile-time constants, compile-time evaluable functions, and raw pointers.

This means that, like macros, `const fn` will be evaluated at compile-time. 
Armed with this knowledge it is straight forward to write a rust function that will pad an array
at compile-time:

# Final code

The code is very simple. Take an array of type `T` and length `N`. 
Append a `pad` value until the result is `M` long.

Because this is a `const` function we can use it to create static variables.

```rust
/// pad an array
/// * `N` - size of the `input`` array
/// * `M` - size of the `final`` array
/// * `T` - type of the array
/// * `pad` - value to use for padding
///
/// in most cases `T` can be omitted and will be inferred by the compiler.
///
/// `N` can be omitted and will be inferred if `#![feature(generic_arg_infer)]`
/// is set.
const fn pad_array<const N: usize, const M: usize, T>(input: [T; N], pad: T) -> [T; M]
where
    T: Copy,
{
    let mut result = [pad; M];
    let mut i = 0;
    while i < input.len() {
        result[i] = input[i];
        i += 1;
    }
    result
}

/// The type argument `T` will be inferred by the compiler
/// without `#![feature(generic_arg_infer)]` ne have to pass the sizes explicitly
pub static DATA_1: [u8; 32] = pad_array::<11,32,_>( *b"Hello World", 0u8);

#![feature(generic_arg_infer)]
/// with `#![feature(generic_arg_infer)]` all sizes will be inferred
pub static DATA_2: [u8; 32] = pad_array::<_,_,_>( *b"Hello World", 0u8);

```

Yes, it is very ugly. And calling it is even uglier. At least, on nightly, with `#![feature(generic_arg_infer)]`
enabled we can omit all the type and size parameters. 

Maybe this could be nicer with macros? Lets see in the feature when I have more time :)

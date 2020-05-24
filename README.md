# Polymorph-lib

This is a header-only library that provides various functionality for randomization on compile-time, in a convenient to use manner that is easy to integrate without any external dependencies or runtime cost.

This is *not* a polymorphic code engine, since it doesn't change the code signature every time it runs, but it takes inspiration from the concept of polymorphic code. You can, however, simulate a polymorphic code engine by recompiling your program each time you would like to run it.

## Use-cases

* **binary fingerprinting** -- you could distribute a different but functionally equivalent binary to multiple people so that they all have a unique executable, and keep track of who received what binary. If your program gets leaked, you can trace the leaker back using the original binary.
* **signature evasion** -- signatures can be generated by hashing files or memory. Code that involves compile-time randomization can evade signature detection between different compiles.
* **non-deterministic algorithms** -- some efficient algorithms are non-deterministic and involve random numbers. Using these at compile-time is made possible through the usage of compile-time random number generators.
* **experimentation** -- it can be useful to ensure that different permutations of code are equivalent through random sampling, as can be done by compiling the program many times with randomization and testing whether they work.

## Usage

1. Put `include/polymorph-lib.h` somewhere in your project.
2. Include it to your project through `#include "polymorph-lib.h"` or some related include statement
3. Use randomized functions. Some major ones are:
- `poly_random(n)` - get a random number between 0 (inclusive) and n (exclusive)
- `poly_junk()` - make junk code that doesn't do anything
- `poly_random_order(f1,f2)` - run the functions `f1` and `f2` in some random order
- `poly_random_chance(c,f)` - random chance to call function `f` -- approximately every `c` distinct calls will call `f` once
- `poly_int()` / `uint()` / `ll()` / `ull()` / `float()` / `double()` - random value of that data type (floating point types range from 0.0 to 1.0).
- `poly_normal(sigma,mu)` - generate a random floating point value following a normal distribution, using sigma and mu values.
4. Compile with some level of optimization (so that redundant branching is removed)

If you want to either manually set a specific seed, or generate a seed using an external program, you can use set the macro `__POLY_RANDOM_SEED__` as in `-D __POLY_RANDOM_SEED__=1234567890ull`. This is optional, and if this is unspecified a seed will be automatically generated based off the current time.

## Details

Before I describe the details of the random number generator, I should delve into the source of entropy used. The GCC supports `__DATE__` and `__TIME__` macros which change on each compile (assuming the compiles aren't made within the same second). We combine those into an integer, that acts as our initial seed.

We then make use of a counter-based-psuedo-random-number-generator (CBPRNG). Normal PRNGs will use usually use their own output as the state to use for their next calculations. Because we are dealing with compile-time calculations this is easier said than done, and it's easier if our state is an available macro like `__COUNTER__` which will automatically increment each time it is referenced. A good fit for this is the [Widynski's Squares](https://arxiv.org/abs/2004.06278) CBPRNG. Because this is all done as a `constexpr` it can be calculated at compile time.

Now we want to turn these compile-time random variables into different code. We make different branches of code (via `if` or `case` statements) with their conditional based on these random variables, and then rely on compiler optimizations to eliminate unused branches.

It's important to note that using functions this way would only evaluate the branch elimination once, and go with that. This means that any call to `poly_junk()` would produce the same code no matter when or where it was used in the program (though it would change on runtime) even when inlined. We instead use `#define` macros for this purpose, so that the preprocessor replaces `poly_junk()` with the corresponding code, which allows the compiler to evaluate each branch separately and results in different code output each time.

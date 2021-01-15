# Valuable Value Virtual Machine

A virtual machine intended as a compilation target for dynamically typed languages whose value set is a superset of the [valuable values](https://github.com/AljoschaMeyer/valuable-value) that also includes first-class functions of fixed arity (closures over mutable environments to be precise) and asynchronous functions.

To define the virtual machine, we need three different concepts: the set of runtime *values*, the *instructions* that are executed by the vm, and the *context* in which instructions executed.

## General Definitions

- a `U63` is a natural number between zero and `2^63 - 1` (inclusive)
- a `U64` is a natural number between zero and `2^64 - 1` (inclusive)
- a `I63` is a natural number between `-2^63` and `2^63 - 1` (inclusive)
- A *sequence* is always finite, but not necessarily nonempty. If `s` is a sequence, then `s[k]` is the k-th element (zero-based indexing).

## Execution Context

The *context* in which instructions are executed consists of the following pieces of data:

- the *instruction map*, a partial mapping from `U63`s to individual instructions
- the *current instruction counter*, a `U63`
- the *globals map*, a partial mapping from `U63`s to individual values
- the *globals offset*, a `U63`
- the *environment table*, a mapping from non-zero natural numbers to individual *environments*:
  - an *environment* consists of a *parent* (a natural number), and a sequence of values
- the *stack*, a sequence of *stack frames*:
  - a *stack frame* is a sequence of values
- the *async context stack*, a sequence of *async contexts*:
  - an *async context* consists of a *current call map*, a *pending call queue*, and a *result queue*:
    - a *current call map* is a partial mapping from natural numbers to *current call*:
      - a *current call* consists of *child calls* (a set of natural numbers) and an optional *continuation* (either the dedicated piece of data `None` or a *continuation*):
        - a *continuation* consists of a *instruction index* (a `U63`) and an *environment index* (a non-zero natural number)
    - a *result queue* is a sequence of *async results*:
      - an *async result* consists of a *result* (a value) and a *finished call* (a natural number)
    - a *pending call queue* is a sequence of *pending calls*:
      - a *pending call* consists of a *callee* (a value), *arguments* (a sequence of values), a *parent call* (a natural number), and if the *parent call* is non-zero a *continuation* (as defined in the *current call*)

## Values

All [valuable values](https://github.com/AljoschaMeyer/valuable-value) are vvvm values, but there are additional ones: synchronous built-in functions, synchronous defined functions, asynchronous built-in functions, and asynchronous defined functions. All of these can also be contained in composite valuable values (arrays, sets and maps).

Synchronous functions are values that can be *called*, i.e. they are being supplied with a function-specific number of argument values and then do one of three things:
  - *return* a value
  - *trap*, i.e. abnormally stop the virtual machine, which cannot be detected from within the virtual machine semantics, but is reported to the outside
  - *diverge*, i.e. executing instruction after instruction without ever returning, or indefinitely waiting for an asynchronous function call to complete, neither of which can be detected from either within the virtual machine semantics or from the outside

Synchronous built-in function values are simply arbitrary values that can be distinguished from all other values. Their semantics are defined by the virtual machine implementation. A set of built-in functions that should be provided by every vvvm is given at the end of this document.

Synchronous defined functions are values consisting of the following pieces of data: an *ordinal* (a natural number that is unique to each defined function value), an *environment index* (a natural number that can be used to look up certain values that can be referred to from the instructions that are performed when the function is called) and an *entrypoint* (a `U63` indicating the instruction to start executing when the function is called).

Asynchronous functions consist of the same data as their synchronous counterparts. Multiple asynchronous functions can be called concurrently, and if multiple of them need to wait for external events to occur (e.g. reading data from a network), then the waiting is done in parallel.

The specifics of how defined functions are being executed are explained in the next section.

## VM Semantics

The semantics of the virtual machine are given as a function that takes an execution *context* as input and then performs a single instruction, returning a new execution context. This is being iterated until `context.result` indicates that execution stops.

The different instructions are as follows:

- 

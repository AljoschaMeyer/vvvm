# Valuable Value Virtual Machine

A virtual machine intended as a compilation target for dynamically typed languages whose value set is a superset of the [valuable values](https://github.com/AljoschaMeyer/valuable-value) that also includes first-class functions of fixed arity (closures over mutable environments to be precise).

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
- the *status*, which is exactly one of the following:
  - *done*, consisting of a value
  - *abnormal termination*
  - *continue*

## Values

All [valuable values](https://github.com/AljoschaMeyer/valuable-value) are vvvm values, but there are additional ones: built-in functions, and defined functions. Both of these can also be (transitively) contained in composite valuable values (arrays, sets and maps).

Functions are values that can be *called*, i.e. they are being supplied with a function-specific number of argument values and then do one of three things:
  - *return* a value
  - *trap*, i.e. abnormally stop the virtual machine, which cannot be detected from within the virtual machine semantics, but is reported to the outside
  - *diverge*, i.e. executing instruction after instruction without ever returning, which can neither be detected from within the virtual machine semantics nor from the outside

Built-in function values are simply arbitrary values that can be distinguished from all other values. Their semantics are defined by the virtual machine implementation. A set of built-in functions that should be provided by every vvvm is given at the end of this document.

Defined functions are values consisting of the following pieces of data: an *ordinal* (a natural number that is unique to each defined function value), an *environment index* (a natural number that can be used to look up certain values that can be referred to from the instructions that are performed when the function is called) and an *entrypoint* (a `U63` indicating the instruction to start executing when the function is called).

The specifics of how defined functions are being executed are explained in the next section.

## VM Semantics

The semantics of the virtual machine are given as a function that takes an execution *context* as input and then performs a single instruction, returning a new execution context. This is being iterated until `context.status` indicates that execution stops.

The different instructions are as follows:

-

# Valuable Value Virtual Machine

An instantiation of the [generic unityped virtual machine](https://github.com/AljoschaMeyer/guvm) (guvm), using a superset of the [valuable values](https://github.com/AljoschaMeyer/valuable-value). All mutability stems from mutable closures, collections are immutable.

The deterministic version of the vvvm uses the deterministic guvm, no further nondeterminism is introduced.

## Values

All [valuable values](https://github.com/AljoschaMeyer/valuable-value) are vvvm values, as are the functions provided by a guvm. Functions can be (transitively) contained in composite valuable values (arrays and maps).

The rest of this document lists all the built-in (asynchronous) functions provided by the vvvm. They are grouped into categories which are also recommended name spaces for programming languages running on the vvvm.

The given time complexities on functions are the worst one that a vvvm implementation may provide. An implementation is free to guarantee *better* complexity bounds than those required.

When a function argument is described as having a certain type, but an argument of a different type supplied, the vm must abort execution. When an argument is referred to as a "positive int", but an int less than zero is supplied, the vm must abort execution as well.

## Built-In Functions

### `value`

This module bundles functions which operate on values of any type.

### `value::halt(v)`

Abnormally halts program execution without producing a value. The vvvm should present the argument to the outside world for diagnostic purposes.

### `value::type_of(v)`

Returns a string describing what kind of value the argument is: `"nil"`, `"bool"`, `"float"`, `"int"`, `"array"`, `"map"`, `"function"`, or `"async"`.

```pavo
assert(value::type_of(nil), "nil");
assert(value::type_of(false), "bool");
assert(value::type_of(0.1), "float");
assert(value::type_of(42), "int");
assert(value::type_of([]), "array");
assert(value::type_of({}), "map");
assert(value::type_of(value::type_of), "function");
assert(value::type_of(nb::preemptive_yield), "async");
```

### `value::truthy(v)`

Returns `false` if the argument is either `nil` or `false`, returns `true` otherwise.

```pavo
assert(value::truthy(nil), false);
assert(value::truthy(false), false);
assert(value::truthy(true), true);
assert(value::truthy(0), true);
```

### `value::falsey(v)`

Returns `true` if the argument is either `nil` or `false`, returns `false` otherwise.

```pavo
assert(value::falsey(nil), true);
assert(value::falsey(false), true);
assert(value::falsey(true), false);
assert(value::falsey(0), false);
```

### `order::total`

This module bundles functions for interacting with the total order over all values. This order is a straightforward extension of the total order over all valuable values: synchronous functions are greater than maps, synchronous built-in functions are ordered in the order in which they appear in this document, synchronous closures are greater than synchronous built-in functions, synchronous closures are ordered by ordinal, asynchronous functions are greater than synchronous functions, asynchronous built-in functions are ordered in the order in which they appear in this document, asynchronous closures are greater than asynchronous built-in functions, asynchronous closures are ordered by ordinal.

See `order::total::compare` for examples illustrating how the order works.

### `order::total::compare(v, w)`

Returns `"<"` if `v` is strictly less than `w` in the total order over all values, `"="` if they are equal, and `">"` if `v` is strictly greater than `w`.

```pavo
assert(order::total::compare(0, 1), "<");
assert(order::total::compare(1, 1), "=");
assert(order::total::compare(1, 0), ">");

# `nil` is less than any other value
assert(order::total::compare(nil, false), "<");

# `false` is less than `true`
assert(order::total::compare(false, true), "<");

# bools are less than floats
assert(order::total::compare(true, 0.1), "<");

# floats are ordered numerically, with `NaN` being the greatest and negative
# zero being less than positive zero
assert(order::total::compare(-Inf, -999.0), "<");
assert(order::total::compare(-999.0, -0.0), "<");
assert(order::total::compare(-0.0, 0.0), "<");
assert(order::total::compare(0.0, 999.0), "<");
assert(order::total::compare(999.0, Inf), "<");
assert(order::total::compare(Inf, NaN), "<");

# floats are less than ints
assert(order::total::compare(1.0, 0), "<");

# ints are ordered numerically
assert(order::total::compare(-2, 1), "<");
assert(order::total::compare(-0, 0), "=");

# ints are less than arrays
assert(order::total::compare(99, []), "<");

# arrays are ordered by size first, lexicographically second
assert(order::total::compare([], [1, 1]), "<");
assert(order::total::compare([1], [1, 1]), "<");
assert(order::total::compare([1, 0], [1, 1]), "<");
assert(order::total::compare([1, 1], [1, 1, 0]), "<");
assert(order::total::compare([1, 1], [2, 0]), "<");

# arrays are less than maps
assert(order::total::compare([999], {}), "<");

# maps are ordered as if they were arrays containing their entries as
# two-element arrays, the outer array being sorted by keys:
# for example `{1: "b", 0: "a"}` corresponds to `[[0, "a"], [1, "b"]]`
assert(order::total::compare({}, {0: 90, 1: 80}), "<");
assert(order::total::compare({0: 90, 1: 79}, {0: 90, 1: 80}), "<");
assert(order::total::compare({0: 90, 1: 80}, {0: 90, 1: 80, 2: 0}), "<");
assert(order::total::compare({1: 79}, {0: 90, 1: 80}), "<");

# maps are less than functions
assert(order::total::compare({}, value::type_of), "<");

# built-in synchronous functions are ordered by their position in this document
assert(order::total::compare(value::type_of, value::truthy), "<");

# synchronous functions are less than asynchronous functions
assert(order::total::compare(value::type_of, nb::preemptive_yield), "<");
```

### `order::total::lt(v, w)`

Returns `true` if `v` is strictly less than `w` in the total order over all values, `false` otherwise.

```pavo
assert(order::total::lt(0, 1), true);
assert(order::total::lt(1, 1), false);
assert(order::total::lt(1, 0), false);
```

### `order::total::leq(v, w)`

Returns `true` if `v` is less than or equal to `w` in the total order over all values, `false` otherwise.

```pavo
assert(order::total::leq(0, 1), true);
assert(order::total::leq(1, 1), true);
assert(order::total::leq(1, 0), false);
```

### `order::total::eq(v, w)`

Returns `true` if `v` is equal to `w` in the total order over all values, `false` otherwise.

```pavo
assert(order::total::eq(0, 1), false);
assert(order::total::eq(1, 1), true);
assert(order::total::eq(1, 0), false);
```

### `order::total::geq(v, w)`

Returns `true` if `v` is greater than or equal to `w` in the total order over all values, `false` otherwise.

```pavo
assert(order::total::geq(0, 1), false);
assert(order::total::geq(1, 1), true);
assert(order::total::geq(1, 0), true);
```

### `order::total::gt(v, w)`

Returns `true` if `v` is strictly greater than `w` in the total order over all values, `false` otherwise.

```pavo
assert(order::total::gt(0, 1), false);
assert(order::total::gt(1, 1), false);
assert(order::total::gt(1, 0), true);
```

### `order::total::neq(v, w)`

Returns `true` if `v` is not equal to `w` in the total order over all values, `false` otherwise.

```pavo
assert(order::total::neq(0, 1), true);
assert(order::total::neq(1, 1), false);
assert(order::total::neq(1, 0), true);
```

### `order::total::min(v, w)`

Returns the argument that is less in the total order over all values.

```pavo
assert(order::total::min(0, 1), 0);
assert(order::total::min(1, 1), 1);
assert(order::total::min(1, 0), 0);
```

### `order::total::max(v, w)`

Returns the argument that is greater in the total order over all values.

```pavo
assert(order::total::max(0, 1), 1);
assert(order::total::max(1, 1), 1);
assert(order::total::max(1, 0), 1);
```

### `order::partial`

This module bundles functions for interacting with the meaningful partial order over all values. This order is a straightforward extension of the meaningful partial order over all valuable values: synchronous functions are incomparable to other kinds of values, two synchronous functions are compared via the total order over all values, asynchronous functions are incomparable to other kinds of values, two asynchronous functions are compared via the total order over all values.

See `order::partial::compare` for examples illustrating how the order works.

### `order::partial::compare(v, w)`

Returns `{"ok": "<"}` if `v` is strictly less than `w` in the meaningful partial order over all values, `{"ok": "="}` if they are equal, and `{"ok": ">"}` if `v` is strictly greater than `w`. Returns `{"err": nil}` if `v` and `w` are incomparable.

```pavo
assert(order::partial::compare(0, 1), {"ok": "<"});
assert(order::partial::compare(1, 1), {"ok": "="});
assert(order::partial::compare(1, 0), {"ok": ">"});

# different kinds of values are incomparable
assert(order::partial::compare(42, []), `{"err": nil}`);

# `false` is less than `true`
assert(order::partial::compare(false, true), {"ok": "<"});

# floats are ordered numerically, with `NaN` being the greatest and negative
# zero being less than positive zero
assert(order::partial::compare(-Inf, -999.0), {"ok": "<"});
assert(order::partial::compare(-999.0, -0.0), "<"{"ok": "<"});
assert(order::partial::compare(-0.0, 0.0), {"ok": "<"});
assert(order::partial::compare(0.0, 999.0), {"ok": "<"});
assert(order::partial::compare(999.0, Inf), {"ok": "<"});
assert(order::partial::compare(Inf, NaN), {"ok": "<"});

# ints are ordered numerically
assert(order::partial::compare(-2, 1), {"ok": "<"});
assert(order::partial::compare(-0, 0), {"ok": "="});

# arrays are ordered component-wise
assert(order::partial::compare([], [1, 1]), {"ok": "<"});
assert(order::partial::compare([1, 0], [1, 1]), {"ok": "<"});
assert(order::partial::compare([0, 1], [1, 1]), {"ok": "<"});
assert(order::partial::compare([0, 2], [1, 1]), {"err": nil});
assert(order::partial::compare([2, 0], [1, 1]), {"err": nil});
assert(order::partial::compare([0, 1, 0], [1, 1]), `{"err": nil}`);
assert(order::partial::compare([2, 1], [1, 1]), {"ok": ">"});
assert(order::partial::compare([1, 2], [1, 1]), {"ok": ">"});
assert(order::partial::compare([1, 1, 0], [1, 1]), {"ok": ">"});

# maps are ordered entry-wise
assert(order::partial::compare({}, {0: 90, 1: 80}), {"ok": "<"});
assert(order::partial::compare({0: 90, 1: 79}, {0: 90, 1: 80}), "<");
assert(order::partial::compare({0: 90, 1: 80}, {0: 90, 1: 80, 2: 0}), "<");
assert(order::partial::compare({0: 89, 1: 81}, {0: 90, 1: 80}), `{"err": nil}`);
assert(order::partial::compare({0: 5}, {0: 4, 1: 0}), `{"err": nil}`);

# built-in functions are ordered by their position in this document
assert(order::partial::compare(value::type_of, value::truthy), {"ok": "<"});
assert(order::partial::compare(value::truthy, value::truthy), {"ok": "="});

# asynchronous and synchronous functions are incomparable
assert(order::partial::compare(value::truthy, nb::preemptive_yield), `{"err": nil}`);
```

### `order::partial::lt(v, w)`

Returns `{"ok": true}` if `v` is strictly less than `w` in the meaningful partial order over all values, `{"ok": false}` if `v` is greater than or equal to `w`, `{"err": nil}` if they are incomparable.

```pavo
assert(order::partial::lt(0, 1), {"ok": true});
assert(order::partial::lt(1, 1), {"ok": false});
assert(order::partial::lt(1, 0), {"ok": false});
assert(order::partial::lt(1, true), {"err": nil});
```

### `order::partial::leq(v, w)`

Returns `{"ok": true}` if `v` is less than or equal to `w` in the meaningful partial order over all values, `{"ok": false}` if `v` is strictly greater than `w`, `{"err": nil}` if they are incomparable.

```pavo
assert(order::partial::leq(0, 1), {"ok": true});
assert(order::partial::leq(1, 1), {"ok": true});
assert(order::partial::leq(1, 0), {"ok": false});
assert(order::partial::leq(1, true), {"err": nil});
```

### `order::partial::eq(v, w)`

Returns `{"ok": true}` if `v` is equal to `w` in the meaningful partial order over all values, `{"ok": false}` if one of them strictly greater than the other, `{"err": nil}` if they are incomparable.

```pavo
assert(order::partial::eq(0, 1), {"ok": false});
assert(order::partial::eq(1, 1), {"ok": true});
assert(order::partial::eq(1, 0), {"ok": false});
assert(order::partial::eq(1, true), {"err": nil});
```

### `order::partial::geq(v, w)`

Returns `{"ok": true}` if `v` is greater than or equal to `w` in the meaningful partial order over all values, `{"ok": false}` if `v` is strictly less than `w`, `{"err": nil}` if they are incomparable.

```pavo
assert(order::partial::geq(0, 1), {"ok": false});
assert(order::partial::geq(1, 1), {"ok": true});
assert(order::partial::geq(1, 0), {"ok": true});
assert(order::partial::geq(1, true), {"err": nil});
```

### `order::partial::gt(v, w)`

Returns `{"ok": true}` if `v` is strictly greater than `w` in the meaningful partial order over all values, `{"ok": false}` if `v` is less than or equal to `w`, `{"err": nil}` if they are incomparable.

```pavo
assert(order::partial::gt(0, 1), {"ok": false});
assert(order::partial::gt(1, 1), {"ok": false});
assert(order::partial::gt(1, 0), {"ok": true});
assert(order::partial::gt(1, true), {"err": nil});
```

### `order::partial::neq(v, w)`

Returns `{"ok": true}` if `v` is not equal to `w` in the meaningful partial order over all values, `{"ok": false}` if `v` is equal to `w`, `{"err": nil}` they are incomparable.

```pavo
assert(order::partial::neq(0, 1), {"ok": true});
assert(order::partial::neq(1, 1), {"ok": false});
assert(order::partial::neq(1, 0), {"ok": true});
assert(order::partial::neq(1, true), {"err": nil});
```

### `order::partial::greatest_lower_bound(v, w)`

Returns `{"ok": glb}`, where `glb` is the greatest value less than or equal to both arguments, or {"err": nil} if no such value exists.

```pavo
assert(order::partial::greatest_lower_bound({0: 90}, {0: 89, 1: 80}), {"ok": {0: 89}});
assert(order::partial::greatest_lower_bound(1, true), {"err": nil});
```

### `order::partial::least_upper_bound(v, w)`

Returns `{"ok": gub}`, where `gub` is the least value greater than or equal to both arguments, or {"err": nil} if no such value exists.

```pavo
assert(order::partial::least_upper_bound({0: 90}, {0: 89, 1: 80}), {"ok": {0: 90, 1: 80}});
assert(order::partial::least_upper_bound(1, true), {"err": nil});
```

### `bool`

This module provides functions operating on bools.

### `bool_not(b)`

Computes the [logical negation](https://en.wikipedia.org/wiki/Negation) of the bool `b`.

```pavo
assert(bool_not(false), true);
assert(bool_not(true), false);
```

### `bool_and(b, c)`

Computes the [logical conjunction](https://en.wikipedia.org/wiki/Logical_conjunction) of the bools `b` and `c`.

```pavo
assert(bool_and(false, false), false);
assert(bool_and(true, false), false);
assert(bool_and(false, true), false);
assert(bool_and(true, true), true);
```

### `bool_or(b, c)`

Computes the [logical disjunction](https://en.wikipedia.org/wiki/Logical_disjunction) of the bools `b` and `c`.

```pavo
assert(bool_or(false, false), false);
assert(bool_or(true, false), true);
assert(bool_or(false, true), true);
assert(bool_or(true, true), true);
```

### `bool_if(b, c)`

Computes the [logical implication](https://en.wikipedia.org/wiki/https://en.wikipedia.org/wiki/Material_conditional) of the bools `b` and `c`.

```pavo
assert(bool_if(false, false), true);
assert(bool_if(true, false), false);
assert(bool_if(false, true), true);
assert(bool_if(true, true), true);
```

### `bool_iff(b, c)`

Computes the [logical biimplication](https://en.wikipedia.org/wiki/Logical_biconditional) of the bools `b` and `c`.

```pavo
assert(bool_iff(false, false), true);
assert(bool_iff(true, false), false);
assert(bool_iff(false, true), false);
assert(bool_iff(true, true), true);
```

### `bool_xor(b, c)`

Computes the [logical exclusive disjunction](https://en.wikipedia.org/wiki/Exclusive_or) of the bools `b` and `c`.

```pavo
assert(bool_xor(false, false), false);
assert(bool_xor(true, false), true);
assert(bool_xor(false, true), true);
assert(bool_xor(true, true), false);
```

### Floats

This module provides functions operating on floats. Unless specified otherwise, these functions work according to the IEEE 754 standard.

### `float_add(x, y)`

Adds the float `x` to the float `y`.

```pavo
assert(float_add(1.0, 2.0), 3.0);
assert(float_add(1.0, -2.0), -1.0);
```

### `float_sub(x, y)`

Subtracts the float `y` from the float `x`.

```pavo
assert(float_sub(1.0, 2.0), -1.0);
assert(float_sub(1.0, -2.0), 3.0);
```

### `float_mul(x, y)`

Multiplies the float `x` with the float `y`.

```pavo
assert(float_mul(2.0, 3.0), 6.0);
assert(float_mul(2.0, -3.0), -6.0);
```

### `float_div(x, y)`

Divides the float `x` by the float `y`.

```pavo
assert(float_div(8.0, 3.0) 2.6666666666666665);
assert(float_div(1.0, 0.0), Inf);
assert(float_div(1.0, -0.0), -Inf);
assert(float_div(0.0, 0.0) NaN);
```

### `float_mul_add(x, y, z)`

Computes `x * y + z` on the floats `x`, `y` and `z` with perfect precision and then rounds the result to the nearest float. This is more accurate than (and thus not equivalent to) `float_add(float_mul(x, y), z)`. Also known as fused multiply-add.

```pavo
assert(float_mul_add(1.2, 3.4, 5.6), 9.68);
```

### `float_neg(x)`

Negates the float `x`.

```pavo
assert(float_neg(1.2), -1.2);
assert(float_neg(-1.2), 1.2);
assert(float_neg(0.0), -0.0);
assert(float_neg(Inf), -Inf);
assert(float_neg(NaN), NaN);
```

### `float_floor(x)`

Returns the largest integral float less than or equal to the float `x`.

```pavo
assert(float_floor(1.9), 1.0);
assert(float_floor(1.0), 1.0);
assert(float_floor(-1.1), -2.0);
```

### `float_ceil(x)`

Returns the smallest integral float greater than or equal to the float `x`.

```pavo
assert(float_ceil(1.1), 2.0);
assert(float_ceil(1.0), 1.0);
assert(float_ceil(-1.9), -1.0);
```

### `float_round(x)`

Rounds the float `x` towards the nearest integral float, rounding towards the even one in case of a tie.

```pavo
assert(float_round(1.0), 1.0);
assert(float_round(1.49), 1.0);
assert(float_round(1.51), 2.0);
assert(float_round(1.5), 2.0);
assert(float_round(2.5), 2.0);
```

### `float_trunc(x)`

Returns the integer part of the float `x` as a float.

```pavo
assert(float_trunc(1.0), 1.0);
assert(float_trunc(1.49), 1.0);
assert(float_trunc(1.51), 1.0);
assert(float_trunc(-1.51), -1.0);
```

### `float_fract(x)`

Returns the fractional part of the float `x` (negative for negative `x`).

```pavo
assert(float_fract(1.0), 0.0);
assert(float_fract(1.49), 0.49);
assert(float_fract(1.51), 0.51);
assert(float_fract(-1.51), -0.51);
```

### `float_abs(x)`

Returns the absolute value of the float `x`.

```pavo
assert(float_abs(1.2), 1.2);
assert(float_abs(-1.2), 1.2);
assert(float_abs(0.0), 0.0);
assert(float_abs(-0.0), 0.0);
```

### `float_signum(x)`

Returns `1.0` if the float `x` is greater than zero, `0.0` if it is equal to zero, `-1.0` if it is less than zero. `NaN` returns `NaN` instead.

```pavo
assert(float_signum(99.2), 1.0);
assert(float_signum(-99.2), -1.0);
assert(float_signum(0.0), 0.0);
assert(float_signum(-0.0), 0.0);
assert(float_signum(NaN), NaN);
```

### `float_pow(x, y)`

Raises the float `x` to the power of the float `y`.

```pavo
assert(float_pow(1.2, 3.4), 1.858729691979481);
```

### `float_sqrt(x)`

Computes the square root of the float `x`.

```pavo
assert(float_sqrt(1.2), 1.0954451150103321);
(assert(float_sqrt(-1.0), NaN);
```

### `float_exp(x)`

Returns [e](https://en.wikipedia.org/wiki/E_(mathematical_constant)) to the power of the float `x`.

```pavo
assert(float_exp(1.2), 3.3201169227365472);
```

### `float_exp2(x)`

Returns 2.0 to the power of the float `x`.

```pavo
assert(float_exp2(1.2), 2.2973967099940698);
```

### `float_ln(x)`

Returns the [natural logarithm](https://en.wikipedia.org/wiki/Natural_logarithm) of the float `x`.

```pavo
assert(float_ln(1.2), 0.1823215567939546);
```

### `float_log2(x)`

Returns the [binary logarithm](https://en.wikipedia.org/wiki/Binary_logarithm) of the float `x`.

```pavo
assert(float_log2(1.2), 0.2630344058337938);
```

### `float_log10(x)`

Returns the base 10 logarithm of the float `x`.

```pavo
assert(float_log10(1.2), 0.07918124604762482);
```

### `float_hypot(x, y)`

Calculates the length of the hypotenuse of a right-angle triangular given legs of the float lengths `x` and `y`.

```pavo
assert(float_hypot(1.2, 3.4), 3.605551275463989);
assert(float_hypot(1.2, -3.4), 3.605551275463989);
assert(float_hypot(-1.2, 3.4), 3.605551275463989);
assert(float_hypot(-1.2, -3.4), 3.605551275463989);
```

### `float_sin(x)`

Computes the sine of the float `x` (in radians).

```pavo
assert(float_sin(1.2), 0.9320390859672263);
```

### `float_cos(x)`

Computes the cosine of the float `x` (in radians).

```pavo
assert(float_cos(1.2), 0.3623577544766736);
```

### `float_tan(x)`

Computes the tangent of the float `x` (in radians).

```pavo
assert(float_tan(1.2), 2.5721516221263188);
```

### `float_asin(x)`

Computes the arcsine of the float `x`, in radians in the range `[-pi/2, pi/2]`. Returns `NaN` if `x` is outside the range `[-1, 1]`.

```pavo
assert(float_asin(0.8), 0.9272952180016123);
assert(float_asin(1.2), NaN);
```

### `float_acos(x)`

Computes the arccosine of the float `x`, in radians in the range `[-pi/2, pi/2]`. Returns `NaN` if `x` is outside the range `[-1, 1]`.

```pavo
assert(float_acos(0.8), 0.6435011087932843);
assert(float_acos(1.2), NaN);
```

### `float_atan(x)`

Computes the arctangent of the float `x`, in radians in the range `[-pi/2, pi/2]`.

```pavo
assert(float_atan(1.2), 0.8760580505981934);
```

### `float_atan2(x, y)`

Computes [atan2](https://en.wikipedia.org/wiki/Atan2) of the float `x` and the float `y`.

```pavo
assert(float_atan2(1.2, 3.4), 0.3392926144540447);
```

### `float_exp_m1(x)`

Returns [e](https://en.wikipedia.org/wiki/E_(mathematical_constant)) to the power of the float `x`, minus one, i.e. `(e^x) - 1`.

```pavo
assert(float_exp_m1(1.2), 2.3201169227365472);
```

### `float_ln_1p(x)`

Returns the [natural logarithm](https://en.wikipedia.org/wiki/Natural_logarithm) of one plus the float `x`, i.e. `ln(1 + x)`

```pavo
assert(float_ln_1p(1.2), 0.7884573603642702);
```

### `float_sinh(x)`

Computes the [hyperbolic sine](https://en.wikipedia.org/wiki/Hyperbolic_function) of the float `x`.

```pavo
assert(float_sinh(1.2), 1.5094613554121725);
```

### `float_cosh(x)`

Computes the [hyperbolic cosine](https://en.wikipedia.org/wiki/Hyperbolic_function) of the float `x`.

```pavo
assert(float_cosh(1.2), 1.8106555673243747);
```

### `float_tanh(x)`

Computes the [hyperbolic tangent](https://en.wikipedia.org/wiki/Hyperbolic_function) of the float `x`.

```pavo
assert(float_tanh(1.2), 0.8336546070121552);
```

### `float_asinh(x)`

Computes the [inverse hyperbolic sine](https://en.wikipedia.org/wiki/Inverse_hyperbolic_functions) of the float `x`.

```pavo
assert(float_asinh(1.2), 1.015973134179692);
```

### `float_acosh(x)`

Computes the [inverse hyperbolic cosine](https://en.wikipedia.org/wiki/Inverse_hyperbolic_functions) of the float `x`.

```pavo
assert(float_acosh(1.2), 0.6223625037147785);
```

### `float_atanh(x)`

Computes the [inverse hyperbolic tangent](https://en.wikipedia.org/wiki/Inverse_hyperbolic_functions) of the float `x`.

```pavo
assert(float_atanh(0.8), 1.0986122886681098);
assert(float_atanh(1.2), NaN);
```

### `float_is_normal(x)`

Returns `true` if the float `x` is neither zero, an infinity, `NaN` or [subnormal](https://en.wikipedia.org/wiki/Denormal_number), false otherwise.

```pavo
assert(float_is_normal(1.0), true);
assert(float_is_normal(1.0e-308), false); # subnormal
assert(float_is_normal(0.0), false);
assert(float_is_normal(Inf), false);
assert(float_is_normal(-Inf), false);
assert(float_is_normal(NaN), false);
```

### `float_is_integral(x)`

Returns `true` if the float `x` is a mathematical integer, false otherwise.

```pavo
assert(float_is_integral(1.0), true);
assert(float_is_integral(0.0), true);
assert(float_is_integral(-42.0), true);
assert(float_is_integral(1.1), false);
assert(float_is_integral(Inf), false);
assert(float_is_integral(-Inf), false);
assert(float_is_integral(NaN), false);
```

### `float_to_degrees(x)`

Converts the float `x` from radians to degrees.

```pavo
assert(float_to_degrees(1.2), 68.75493541569878);
```

### `float_to_radians(x)`

Converts the float `x` from degrees to radians.

```pavo
assert(float_to_radians(1.2), 0.020943951023931952);
```

### `float_to_int(x)`

Tries to converts the float `x` to an int `i` by checking whether `float_round(x)` is an integer between `-2^63` and `2^63 - 1`. Returns `{"ok": i}` if it is, `{"err": x}` otherwise.

```pavo
assert(float_to_int(1.0), {"ok": 1});
assert(float_to_int(-1.0), {"ok": -1});
assert(float_to_int(0.0), {"ok": 0});
assert(float_to_int(-0.0), {"ok": 0});
assert(float_to_int(1.9), {"ok": 1});
assert(float_to_int(-1.9), {"ok": -1});
assert(float_to_int(59907199254740993.9), {"ok": 59907199254740990});
assert(float_to_int(Inf), {"err": Inf});
assert(float_to_int(-Inf), {"err": -Inf});
assert(float_to_int(NaN), {"err": NaN});
```

### `float_from_int(n)`

Converts the int `n` to a float, using the usual rounding rules if it can not be represented exactly ([round to nearest, ties to even](https://en.wikipedia.org/wiki/Rounding#Round_half_to_even)).

```pavo
assert(float_from_int(0) 0.0);
assert(float_from_int(1) 1.0);
assert(float_from_int(-1) -1.0);
assert(float_from_int(9007199254740993) 9007199254740992.0);
assert(float_from_int(-9007199254740993) -9007199254740992.0);
```

### `float_to_bits(x)`

Returns the int with the same bit pattern as the bit pattern of the float `x`. The sign bit of the float is the most significant bit of the resulting int, the least significant bit of the float's mantissa is the least significant bit of the resulting int.

`NaN` counts as a bit pattern consisting only of 1s.

```pavo
assert(float_to_bits(1.2), 4608083138725491507);
assert(float_to_bits(-1.2), -4615288898129284301);
assert(float_to_bits(0.0), 0);
assert(float_to_bits(-0.0), -9223372036854775808);
assert(float_to_bits(NaN), -1);
```

### `bits_to_float(n)`

Returns the float with the same bit pattern as the bit pattern of the int `n`. The most significant bit of `n` is the sign bit of the float, the least significant bit of `n` is the least significant bit of the float's mantissa.

All bit patterns representing an IEEE 754 not-a-number return `NaN`.

```pavo
assert(bits_to_float(42), 2.08e-322);
assert(bits_to_float(-1), NaN);
assert(bits_to_float(9221120237041090560), NaN);
```









TODO

- float
- int
- array
- map
- function
- preemptive_yield
- builders, iterators

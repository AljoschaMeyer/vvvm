# Valuable Value Virtual Machine

An instantiation of the [generic unityped virtual machine](https://github.com/AljoschaMeyer/guvm) (guvm), using a superset of the [valuable values](https://github.com/AljoschaMeyer/valuable-value). All mutability stems from mutable closures, collections are immutable.

The deterministic version of the vvvm uses the deterministic guvm, no further nondeterminism is introduced.

## Values

All [valuable values](https://github.com/AljoschaMeyer/valuable-value) are vvvm values, as are the functions provided by a guvm. Functions can be (transitively) contained in composite valuable values (arrays and maps).

The rest of this document lists all the built-in (asynchronous) functions provided by the vvvm. They are grouped into categories which are also recommended name spaces for programming languages running on the vvvm.

The given time complexities on functions are the worst one that a vvvm implementation may provide. An implementation is free to guarantee *better* complexity bounds than those required.

When a function argument is described as having a certain type, but an argument of a different type supplied, the vm must abort execution. When an argument is referred to as a "positive int", but an int less than zero is supplied, the vm must abort execution. When an argument is referred to as a "nonzero int", but `0` it is supplied, the vm must abort execution.

When a function argument is referred to as an "index", the vm must abort execution if it is not an int, strictly less than `0`, or greater or equal to the number of elements in the first collection argument of the function (array or map). Indices are effectively a numbering of the elements in a collection, starting at 0.

When a function argument is referred to as a "position", the vm must abort execution if it is not an int, strictly less than `0`, or strictly greater than the number of elements in the first collection argument of the function (array or map). Positions effectively sit between the elements of a collection, with position 0 being in front of the first element, and the last position being behind the last element.

When a function is said to return an "object", it returns a closure taking two arguments. The first argument indicates which method of the so-called object to invoke, the second argument is an array containing the arguments to that method. The documentation then lists the available methods and which arguments they take. Specifying an invalid method or not passing the correct number of arguments in the array aborts execution.

When something is said to be a string, this refers to an array containing the bytes of a utf-8 string as ints.

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

# arrays are ordered lexicographically
assert(order::total::compare([], [1, 1]), "<");
assert(order::total::compare([1], [1, 1]), "<");
assert(order::total::compare([1, 0], [1, 1]), "<");
assert(order::total::compare([1, 1], [1, 1, 0]), "<");
assert(order::total::compare([1, 1], [2]), "<");

# arrays are less than maps
assert(order::total::compare([999], {}), "<");

# maps are ordered as if they were arrays containing their entries as
# two-element arrays, the outer array being sorted by keys:
# for example `{1: "b", 0: "a"}` corresponds to `[[0, "a"], [1, "b"]]`
assert(order::total::compare({}, {0: 90, 1: 80}), "<");
assert(order::total::compare({0: 90, 1: 79}, {0: 90, 1: 80}), "<");
assert(order::total::compare({0: 90, 1: 80}, {0: 90, 1: 80, 2: 0}), "<");
assert(order::total::compare({0: 90, 1: 80}, {1: 79}), "<");

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

# arrays are ordered component-wise, with the smaller array conceptually being
# right-padded with values that are less than any real value
assert(order::partial::compare([], [1, 1]), {"ok": "<"});
assert(order::partial::compare([1, 0], [1, 1]), {"ok": "<"});
assert(order::partial::compare([0, 1], [1, 1]), {"ok": "<"});
assert(order::partial::compare([0, 1, 0], [1, 1]), `{"ok": "<"}`);
assert(order::partial::compare([0, 2], [1, 1]), {"err": nil});
assert(order::partial::compare([2, 0], [1, 1]), {"err": nil});
assert(order::partial::compare([2], [1, 1]), {"err": nil});
assert(order::partial::compare([2, 1], [1, 1]), {"ok": ">"});
assert(order::partial::compare([1, 2], [1, 1]), {"ok": ">"});
assert(order::partial::compare([1, 1, 0], [1, 1]), {"ok": ">"});

# maps are ordered entry-wise, if one map has a key the other map does not, the
# is considered to map that key to a value less than any real value
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

### `float::add(x, y)`

Adds the float `x` to the float `y`.

```pavo
assert(float::add(1.0, 2.0), 3.0);
assert(float::add(1.0, -2.0), -1.0);
```

### `float::sub(x, y)`

Subtracts the float `y` from the float `x`.

```pavo
assert(float::sub(1.0, 2.0), -1.0);
assert(float::sub(1.0, -2.0), 3.0);
```

### `float::mul(x, y)`

Multiplies the float `x` with the float `y`.

```pavo
assert(float::mul(2.0, 3.0), 6.0);
assert(float::mul(2.0, -3.0), -6.0);
```

### `float::div(x, y)`

Divides the float `x` by the float `y`.

```pavo
assert(float::div(8.0, 3.0) 2.6666666666666665);
assert(float::div(1.0, 0.0), Inf);
assert(float::div(1.0, -0.0), -Inf);
assert(float::div(0.0, 0.0) NaN);
```

### `float::mul_add(x, y, z)`

Computes `x * y + z` on the floats `x`, `y` and `z` with perfect precision and then rounds the result to the nearest float. This is more accurate than (and thus not equivalent to) `float::add(float::mul(x, y), z)`. Also known as fused multiply-add.

```pavo
assert(float::mul_add(1.2, 3.4, 5.6), 9.68);
```

### `float::neg(x)`

Negates the float `x`.

```pavo
assert(float::neg(1.2), -1.2);
assert(float::neg(-1.2), 1.2);
assert(float::neg(0.0), -0.0);
assert(float::neg(Inf), -Inf);
assert(float::neg(NaN), NaN);
```

### `float::floor(x)`

Returns the largest integral float less than or equal to the float `x`.

```pavo
assert(float::floor(1.9), 1.0);
assert(float::floor(1.0), 1.0);
assert(float::floor(-1.1), -2.0);
```

### `float::ceil(x)`

Returns the smallest integral float greater than or equal to the float `x`.

```pavo
assert(float::ceil(1.1), 2.0);
assert(float::ceil(1.0), 1.0);
assert(float::ceil(-1.9), -1.0);
```

### `float::round(x)`

Rounds the float `x` towards the nearest integral float, rounding towards the even one in case of a tie.

```pavo
assert(float::round(1.0), 1.0);
assert(float::round(1.49), 1.0);
assert(float::round(1.51), 2.0);
assert(float::round(1.5), 2.0);
assert(float::round(2.5), 2.0);
```

### `float::trunc(x)`

Returns the integer part of the float `x` as a float.

```pavo
assert(float::trunc(1.0), 1.0);
assert(float::trunc(1.49), 1.0);
assert(float::trunc(1.51), 1.0);
assert(float::trunc(-1.51), -1.0);
```

### `float::fract(x)`

Returns the fractional part of the float `x` (negative for negative `x`).

```pavo
assert(float::fract(1.0), 0.0);
assert(float::fract(1.49), 0.49);
assert(float::fract(1.51), 0.51);
assert(float::fract(-1.51), -0.51);
```

### `float::abs(x)`

Returns the absolute value of the float `x`.

```pavo
assert(float::abs(1.2), 1.2);
assert(float::abs(-1.2), 1.2);
assert(float::abs(0.0), 0.0);
assert(float::abs(-0.0), 0.0);
```

### `float::signum(x)`

Returns `1.0` if the float `x` is greater than zero, `0.0` if it is equal to zero, `-1.0` if it is less than zero. `NaN` returns `NaN` instead.

```pavo
assert(float::signum(99.2), 1.0);
assert(float::signum(-99.2), -1.0);
assert(float::signum(0.0), 0.0);
assert(float::signum(-0.0), 0.0);
assert(float::signum(NaN), NaN);
```

### `float::pow(x, y)`

Raises the float `x` to the power of the float `y`.

```pavo
assert(float::pow(1.2, 3.4), 1.858729691979481);
```

### `float::sqrt(x)`

Computes the square root of the float `x`.

```pavo
assert(float::sqrt(1.2), 1.0954451150103321);
(assert(float::sqrt(-1.0), NaN);
```

### `float::exp(x)`

Returns [e](https://en.wikipedia.org/wiki/E_(mathematical_constant)) to the power of the float `x`.

```pavo
assert(float::exp(1.2), 3.3201169227365472);
```

### `float::exp2(x)`

Returns 2.0 to the power of the float `x`.

```pavo
assert(float::exp2(1.2), 2.2973967099940698);
```

### `float::ln(x)`

Returns the [natural logarithm](https://en.wikipedia.org/wiki/Natural_logarithm) of the float `x`.

```pavo
assert(float::ln(1.2), 0.1823215567939546);
```

### `float::log2(x)`

Returns the [binary logarithm](https://en.wikipedia.org/wiki/Binary_logarithm) of the float `x`.

```pavo
assert(float::log2(1.2), 0.2630344058337938);
```

### `float::log10(x)`

Returns the base 10 logarithm of the float `x`.

```pavo
assert(float::log10(1.2), 0.07918124604762482);
```

### `float::hypot(x, y)`

Calculates the length of the hypotenuse of a right-angle triangular given legs of the float lengths `x` and `y`.

```pavo
assert(float::hypot(1.2, 3.4), 3.605551275463989);
assert(float::hypot(1.2, -3.4), 3.605551275463989);
assert(float::hypot(-1.2, 3.4), 3.605551275463989);
assert(float::hypot(-1.2, -3.4), 3.605551275463989);
```

### `float::sin(x)`

Computes the sine of the float `x` (in radians).

```pavo
assert(float::sin(1.2), 0.9320390859672263);
```

### `float::cos(x)`

Computes the cosine of the float `x` (in radians).

```pavo
assert(float::cos(1.2), 0.3623577544766736);
```

### `float::tan(x)`

Computes the tangent of the float `x` (in radians).

```pavo
assert(float::tan(1.2), 2.5721516221263188);
```

### `float::asin(x)`

Computes the arcsine of the float `x`, in radians in the range `[-pi/2, pi/2]`. Returns `NaN` if `x` is outside the range `[-1, 1]`.

```pavo
assert(float::asin(0.8), 0.9272952180016123);
assert(float::asin(1.2), NaN);
```

### `float::acos(x)`

Computes the arccosine of the float `x`, in radians in the range `[-pi/2, pi/2]`. Returns `NaN` if `x` is outside the range `[-1, 1]`.

```pavo
assert(float::acos(0.8), 0.6435011087932843);
assert(float::acos(1.2), NaN);
```

### `float::atan(x)`

Computes the arctangent of the float `x`, in radians in the range `[-pi/2, pi/2]`.

```pavo
assert(float::atan(1.2), 0.8760580505981934);
```

### `float::atan2(x, y)`

Computes [atan2](https://en.wikipedia.org/wiki/Atan2) of the float `x` and the float `y`.

```pavo
assert(float::atan2(1.2, 3.4), 0.3392926144540447);
```

### `float::exp_m1(x)`

Returns [e](https://en.wikipedia.org/wiki/E_(mathematical_constant)) to the power of the float `x`, minus one, i.e. `(e^x) - 1`.

```pavo
assert(float::exp_m1(1.2), 2.3201169227365472);
```

### `float::ln_1p(x)`

Returns the [natural logarithm](https://en.wikipedia.org/wiki/Natural_logarithm) of one plus the float `x`, i.e. `ln(1 + x)`

```pavo
assert(float::ln_1p(1.2), 0.7884573603642702);
```

### `float::sinh(x)`

Computes the [hyperbolic sine](https://en.wikipedia.org/wiki/Hyperbolic_function) of the float `x`.

```pavo
assert(float::sinh(1.2), 1.5094613554121725);
```

### `float::cosh(x)`

Computes the [hyperbolic cosine](https://en.wikipedia.org/wiki/Hyperbolic_function) of the float `x`.

```pavo
assert(float::cosh(1.2), 1.8106555673243747);
```

### `float::tanh(x)`

Computes the [hyperbolic tangent](https://en.wikipedia.org/wiki/Hyperbolic_function) of the float `x`.

```pavo
assert(float::tanh(1.2), 0.8336546070121552);
```

### `float::asinh(x)`

Computes the [inverse hyperbolic sine](https://en.wikipedia.org/wiki/Inverse_hyperbolic_functions) of the float `x`.

```pavo
assert(float::asinh(1.2), 1.015973134179692);
```

### `float::acosh(x)`

Computes the [inverse hyperbolic cosine](https://en.wikipedia.org/wiki/Inverse_hyperbolic_functions) of the float `x`.

```pavo
assert(float::acosh(1.2), 0.6223625037147785);
```

### `float::atanh(x)`

Computes the [inverse hyperbolic tangent](https://en.wikipedia.org/wiki/Inverse_hyperbolic_functions) of the float `x`.

```pavo
assert(float::atanh(0.8), 1.0986122886681098);
assert(float::atanh(1.2), NaN);
```

### `float::is_normal(x)`

Returns `true` if the float `x` is neither zero, an infinity, `NaN` or [subnormal](https://en.wikipedia.org/wiki/Denormal_number), false otherwise.

```pavo
assert(float::is_normal(1.0), true);
assert(float::is_normal(1.0e-308), false); # subnormal
assert(float::is_normal(0.0), false);
assert(float::is_normal(Inf), false);
assert(float::is_normal(-Inf), false);
assert(float::is_normal(NaN), false);
```

### `float::to_degrees(x)`

Converts the float `x` from radians to degrees.

```pavo
assert(float::to_degrees(1.2), 68.75493541569878);
```

### `float::to_radians(x)`

Converts the float `x` from degrees to radians.

```pavo
assert(float::to_radians(1.2), 0.020943951023931952);
```

### `float::to_int(x)`

Tries to converts the float `x` to an int `i` by checking whether `float::round(x)` is an integer between `-2^63` and `2^63 - 1`. Returns `{"ok": i}` if it is, `{"err": x}` otherwise.

```pavo
assert(float::to_int(1.0), {"ok": 1});
assert(float::to_int(-1.0), {"ok": -1});
assert(float::to_int(0.0), {"ok": 0});
assert(float::to_int(-0.0), {"ok": 0});
assert(float::to_int(1.9), {"ok": 1});
assert(float::to_int(-1.9), {"ok": -1});
assert(float::to_int(59907199254740993.9), {"ok": 59907199254740990});
assert(float::to_int(Inf), {"err": Inf});
assert(float::to_int(-Inf), {"err": -Inf});
assert(float::to_int(NaN), {"err": NaN});
```

### `float::from_int(n)`

Converts the int `n` to a float, using the usual rounding rules if it can not be represented exactly ([round to nearest, ties to even](https://en.wikipedia.org/wiki/Rounding#Round_half_to_even)).

```pavo
assert(float::from_int(0) 0.0);
assert(float::from_int(1) 1.0);
assert(float::from_int(-1) -1.0);
assert(float::from_int(9007199254740993) 9007199254740992.0);
assert(float::from_int(-9007199254740993) -9007199254740992.0);
```

### `float::to_bits(x)`

Returns the int with the same bit pattern as the bit pattern of the float `x`. The sign bit of the float is the most significant bit of the resulting int, the least significant bit of the float's mantissa is the least significant bit of the resulting int.

`NaN` counts as a bit pattern consisting only of 1s.

```pavo
assert(float::to_bits(1.2), 4608083138725491507);
assert(float::to_bits(-1.2), -4615288898129284301);
assert(float::to_bits(0.0), 0);
assert(float::to_bits(-0.0), -9223372036854775808);
assert(float::to_bits(NaN), -1);
```

### `float::from_bits(n)`

Returns the float with the same bit pattern as the bit pattern of the int `n`. The most significant bit of `n` is the sign bit of the float, the least significant bit of `n` is the least significant bit of the float's mantissa.

All bit patterns representing an IEEE 754 not-a-number return `NaN`.

```pavo
assert(float::from_bits(42), 2.08e-322);
assert(float::from_bits(-1), NaN);
assert(float::from_bits(9221120237041090560), NaN);
```

### `int`

This module provides functions operating on ints.

### `int::signum(n)`

Returns `-1` if the int `n` is less than `0`, `0` if `n` is equal to `0`, `1` if `n` is greater than `0`.

```pavo
assert(int::signum(-42), -1);
assert(int::signum(0), 0);
assert(int::signum(42), 1);
```

### `int::add(n, m)`

Adds the int `n` to the int `m`. The vm aborts execution if an overflow occurs.

```pavo
assert(int::add(1, 2), 3);
assert(int::add(1, -2), -1);
```

### `int::sub(n, m)`

Subtracts the int `m` from the int `n`. The vm aborts execution if an overflow occurs.

```pavo
assert(int::sub(1, 2), -1);
assert(int::sub(1, -2), 3);
```

### `int::mul(n, m)`

Multiplies the int `n` with the int `m`. The vm aborts execution if an overflow occurs.

```pavo
assert(int::mul(2, 3), 6);
assert(int::mul(2, -3), -6);
```

### `int::div(n, m)`

Divides the int `n` by the nonzero int `m`. The vm aborts execution if an overflow occurs.

This computes the quotient of [euclidean division](https://en.wikipedia.org/wiki/Euclidean_division).

```pavo
assert(int::div(8, 3), 2);
assert(int::div(-8, 3), -3);
```

### `int::div_trunc(n, m)`

Divides the int `n` by the nonzero int `m`. The vm aborts execution if an overflow occurs.

This computes the quotient of [truncating division](https://en.wikipedia.org/w/index.php?title=Truncated_division).

```pavo
assert(int::div_trunc(8, 3), 2);
assert(int::div_trunc(-8, 3), -2);
```

### `int::mod(n, m)`

Computes the int `n` modulo the nonzero int `m`. The vm aborts execution if an overflow occurs.

This computes the remainder of [euclidean division](https://en.wikipedia.org/wiki/Euclidean_division).

```pavo
assert(int::mod(8, 3), 2);
assert(int::mod(-8, 3), 1);
```

### `int::mod_trunc(n, m)`

Computes the int `n` modulo the nonzero int `m`. The vm aborts execution if an overflow occurs.

This computes the remainder of [truncating division](https://en.wikipedia.org/w/index.php?title=Truncated_division).

```pavo
assert(int::mod_trunc(8, 3), 2);
assert(int::mod_trunc(-8, 3), -2);
```

### `int::neg(n)`

Negates the int `n`. The vm aborts execution if an overflow occurs.

```pavo
assert(int::neg(42), -42);
assert(int::neg(-42), 42);
assert(int::neg(0), 0);
```

### `int::abs(n)`

Computes the absolute value of the int `n`. The vm aborts execution if an overflow occurs.

```pavo
assert(int::abs(42), 42);
assert(int::abs(-42), 42);
assert(int::abs(0), 0);
```

### `int::pow(n, m)`

Computes the int `n` to the power of the positive int `m`. The vm aborts execution if an overflow occurs.

```pavo
assert(int::pow(2, 3) 8);
assert(int::pow(2, 0), 1);
assert(int::pow(0, 999), 0);
assert(int::pow(1, 999), 1);
assert(int::pow(-1, 999), -1);
```

### `int::check`

Operation on int that check for overflows and return results accordingly.

### `int::check::add(n, m)`

Adds the int `n` to the int `m`. Returns `{"ok": sum}` if no overflow occurred, `{"err": nil}` otherwise.

```pavo
assert(int::check::add(1, 2), {"ok": 3});
assert(int::check::add(1, -2), {"ok": -1});
assert(int::check::add(9223372036854775807, 1), {"err": nil});
```

### `int::check::sub(n, m)`

Subtracts the int `m` from the int `n`. Returns `{"ok": difference}` if no overflow occurred, `{"err": nil}` otherwise.

```pavo
assert(int::check::sub(1, 2), {"ok": -1});
assert(int::check::sub(1, -2), {"ok": 3});
assert(int::check::sub(-9223372036854775808, 1), {"ok": 3});
```

### `int::check::mul(n, m)`

Multiplies the int `n` with the int `m`. Returns `{"ok": product}` if no overflow occurred, `{"err": nil}` otherwise.

```pavo
assert(int::check::mul(2, 3), {"ok": 6});
assert(int::check::mul(2, -3), {"ok": -6});
assert(int::check::mul(2, 9223372036854775807), {"err": nil});
```

### `int::check::div(n, m)`

Divides the int `n` by the nonzero int `m`. Returns `{"ok": quotient}` if no overflow occurred, `{"err": "overflow"}` otherwise.

This computes the quotient of [euclidean division](https://en.wikipedia.org/wiki/Euclidean_division).

```pavo
assert(int::check::div(8, 3), {"ok": 2});
assert(int::check::div(-8, 3), {"ok": -3});
assert(int::check::div(-9223372036854775808, -1), {"err": "overflow"});
```

### `int::check::div_trunc(n, m)`

Divides the int `n` by the nonzero int `m`. Returns `{"ok": quotient}` if no overflow occurred, `{"err": "overflow"}` otherwise.

This computes the quotient of [truncating division](https://en.wikipedia.org/w/index.php?title=Truncated_division).

```pavo
assert(int::check::div_trunc(8, 3), {"ok": 2});
assert(int::check::div_trunc(-8, 3), {"ok": -2});
assert(int::check::div_trunc(-9223372036854775808, -1), {"err": "overflow"});
```

### `int::check::mod(n, m)`

Computes the int `n` modulo the nonzero int `m`. Returns`{"ok": modulus}` if no overflow occurred, `{"err": "overflow"}` otherwise.

This computes the remainder of [euclidean division](https://en.wikipedia.org/wiki/Euclidean_division).

```pavo
assert(int::check::mod(8, 3), {"ok": 2});
assert(int::check::mod(-8, 3), {"ok": 1});
assert(int::check::mod(-9223372036854775808, -1), {"err": "overflow"});
```

### `int::check::mod_trunc(n, m)`

Computes the int `n` modulo the nonzero int `m`. Returns `{"ok": modulus}` if no overflow occurred, `{"err": "overflow"}` otherwise.

This computes the remainder of [truncating division](https://en.wikipedia.org/w/index.php?title=Truncated_division).

```pavo
assert(int::check::mod_trunc(8, 3), {"ok": 2});
assert(int::check::mod_trunc(-8, 3), {"ok": -2});
assert(int::check::mod_trunc(-9223372036854775808, -1), {"err": "overflow"});
```

### `int::check::neg(n)`

Negates the int `n`. Returns `{"ok": negated}` if no overflow occurred, `{"err": nil}` otherwise.

```pavo
assert(int::check::neg(42), {"ok": -42});
assert(int::check::neg(-42), {"ok": 42});
assert(int::check::neg(0), {"ok": 0});
assert(int::check::neg(-9223372036854775808), {"err": nil});
```

### `int::check::abs(n)`

Computes the absolute value of the int `n`. Returns `{"ok": absolute}` if no overflow occurred, `{"err": nil}` otherwise.

```pavo
assert(int::check::abs(42), {"ok": 42});
assert(int::check::abs(-42), {"ok": 42});
assert(int::check::abs(0), {"ok": 0});
assert(int::check::abs(-9223372036854775808), {"err": nil});
```

### `int::check::pow(n, m)`

Computes the int `n` to the power of the positive int `m`. Returns `{"ok": power}` if no overflow occurred, `{"err": nil}` otherwise.

```pavo
assert(int::check::pow(2, 3) {"ok": 8});
assert(int::check::pow(2, 0), {"ok": 1});
assert(int::check::pow(0, 999), {"ok": 0});
assert(int::check::pow(1, 999), {"ok": 1});
assert(int::check::pow(-1, 999), {"ok": -1});
assert(int::check::pow(99, 99), {"err": nil});
```

### `int::sat`

Computation on int that saturate at the numeric bounds instead of overflowing.

### `int::sat::add(n, m)`

Adds the int `n` to the int `m`, saturating at the numeric bounds instead of overflowing.

```pavo
assert(int::sat::add(1, 2), 3);
assert(int::sat::add(1, -2), -1);
assert(int::sat::add(9223372036854775807, 1), 9223372036854775807);
assert(int::sat::add(-9223372036854775808, -1), -9223372036854775808);
```

### `int::sat::sub(n, m)`

Subtracts the int `n` from the int `m`, saturating at the numeric bounds instead of overflowing.

```pavo
assert(int::sat::sub(1, 2), -1);
assert(int::sat::sub(1, -2), 3);
assert(int::sat::sub(-9223372036854775808, 1), -9223372036854775808);
assert(int::sat::sub(9223372036854775807, -1), 9223372036854775807);
```

### `int::sat::mul(n, m)`

Multiplies the int `n` with the int `m`, saturating at the numeric bounds instead of overflowing.

```pavo
assert(int::sat::mul(2, 3), 6);
assert(int::sat::mul(2, -3), -6);
assert(int::sat::mul(9223372036854775807, 2), 9223372036854775807);
assert(int::sat::mul(-9223372036854775808, 2), -9223372036854775808);
```

### `int::sat::pow(n, m)`

Computes the int `n` to the power of the positive int `m`, saturating at the numeric bounds instead of overflowing.

```pavo
assert(int::sat::pow(2, 3), 8);
assert(int::sat::pow(2, 0), 1);
assert(int::sat::pow(0, 999), 0);
assert(int::sat::pow(1, 999), 1);
assert(int::sat::pow(-1, 999), -1);
assert(int::sat::pow(99, 99), 9223372036854775807);
assert(int::sat::pow(-99, 99), -9223372036854775808);
```

### `int::wrap`

Computations on ints that wrap around numeric bounds instead of overflowing.

### `int::wrap::add(n, m)`

Adds the int `n` to the int `m`, wrapping around the numeric bounds instead of overflowing.

```pavo
assert(int::wrap::add(1, 2), 3);
assert(int::wrap::add(9223372036854775807, 1), -9223372036854775808);
assert(int::wrap::add(-9223372036854775808, -1), 9223372036854775807);
```

### `int::wrap::sub(n, m)`

Subtracts the int `n` from the int `m`, wrapping around the numeric bounds instead of overflowing.

```pavo
assert(int::wrap::sub(1, 2), -1);
assert(int::wrap::sub(-9223372036854775808, 1), 9223372036854775807);
assert(int::wrap::sub(9223372036854775807, -1), -9223372036854775808);
```

### `int::wrap::mul(n, m)`

Muliplies the int `n` with the int `m`, wrapping around the numeric bounds instead of overflowing.

```pavo
assert(int::wrap::mul(2, 3), 6);
assert(int::wrap::mul(9223372036854775807, 2), -2);
assert(int::wrap::mul(9223372036854775807, -2), 2);
assert(int::wrap::mul(-9223372036854775808, 2), 0);
assert(int::wrap::mul(-9223372036854775808, -2), 0);
```

### `int::wrap::div(n, m)`

Divides the nonzero int `n` by the int `m`, wrapping around the numeric bounds instead of overflowing.

This computes the quotient of [euclidean division](https://en.wikipedia.org/wiki/Euclidean_division).

```pavo
assert(int::wrap::div(8, 3), 2);
assert(int::wrap::div(-8, 3), -3);
assert(int::wrap::div(-9223372036854775808, -1), -9223372036854775808);
```

### `int::wrap::div_trunc(n, m)`

Divides the nonzero int `n` by the int `m`, wrapping around the numeric bounds instead of overflowing.

This computes the quotient of [truncating division](https://en.wikipedia.org/w/index.php?title=Truncated_division).

```pavo
assert(int::wrap::div_trunc(8, 3), 2);
assert(int::wrap::div_trunc(-8, 3), -2);
assert(int::wrap::div_trunc(-9223372036854775808, -1), -9223372036854775808);
```

### `int::wrap::mod(n, m)`

Computes the int `n` modulo the nonzero int `m`, wrapping around the numeric bounds instead of overflowing.

This computes the remainder of [euclidean division](https://en.wikipedia.org/wiki/Euclidean_division).

```pavo
assert(int::wrap::mod(8, 3), 2);
assert(int::wrap::mod(-8, 3), 1);
assert(int::wrap::mod(-9223372036854775808, -1), 0);
```

### `int::wrap::mod_trunc(n, m)`

Computes the int `n` modulo the nonzero int `m`, wrapping around the numeric bounds instead of overflowing.

This computes the remainder of [truncating division](https://en.wikipedia.org/w/index.php?title=Truncated_division).

```pavo
assert(int::wrap::mod_trunc(8, 3), 2);
assert(int::wrap::mod_trunc(-8, 3), -2);
assert(int::wrap::mod_trunc(-9223372036854775808, -1), 0);
```

### `int::wrap::neg(n)`

Negates the int `n`, wrapping around the numeric bounds instead of overflowing.

```pavo
assert(int::wrap::neg(42), -42);
assert(int::wrap::neg(-42) 42);
assert(int::wrap::neg(0), 0);
assert(int::wrap::neg(-9223372036854775808), -9223372036854775808);
```

### `int::wrap::abs(n)`

Returns the absolute value of the int `n`, wrapping around the numeric bounds instead of overflowing.

```pavo
assert(int::wrap::abs(42), 42);
assert(int::wrap::abs(-42), 42);
assert(int::wrap::abs(0), 0);
assert(int::wrap::abs(-9223372036854775808), -9223372036854775808);
```

### `int::wrap::pow(n, m)`

Computes the int `n` to the power of the positive int `m`, wrapping around the numeric bounds instead of overflowing.

```pavo
assert(int::wrap::pow(2, 3), 8);
assert(int::wrap::pow(2, 0), 1);
assert(int::wrap::pow(0, 999), 0);
assert(int::wrap::pow(1, 999), 1);
assert(int::wrap::pow(-1, 999), -1);
assert(int::wrap::pow(99, 99), -7394533151961528133);
```

### `int::bit`

This module contains functions working on the bit representation of ints.

### `int::bit::count_ones(n)`

Returns the number of ones in the binary representation of the int `n`.

```pavo
assert(int::bit::count_ones(126), 6);
```

### `int::bit::count_zeros(n)`

Returns the number of zeros in the binary representation of the int `n`.

```pavo
assert(int::bit::count_zeros(126), 58);
```

### `int::bit::leading_ones(n)`

Returns the number of leading ones in the binary representation of the int `n`.

```pavo
assert(int::bit::leading_ones(-4611686018427387904), 2);
```

### `int::bit::leading_zeros(n)`

Returns the number of leading zeros in the binary representation of the int `n`.

```pavo
assert(int::bit::leading_zeros(13), 60);
```

### `int::bit::trailing_ones(n)`

Returns the number of trailing ones in the binary representation of the int `n`.

```pavo
assert(int::bit::trailing_ones(3), 2);
```

### `int::bit::trailing_zeros(n)`

Returns the number of trailing zeros in the binary representation of the int `n`.

```pavo
assert(int::bit::trailing_zeros(4), 2);
```

### `int::bit::rotate_left(n, by)`

Shifts the bits of the int `n` to the left by the positive int `by`, wrapping the truncated bits to the end of the resulting int.

```pavo
assert(int::bit::rotate_left(0xaa00000000006e1, 12), 0x6e10aa);
```

### `int::bit::rotate_right(n, by)`

Shifts the bits of the int `n` to the right by the positive int `by`, wrapping the truncated bits to the beginning of the resulting int.

```pavo
assert(int::bit::rotate_right(0x6e10aa, 12), 0xaa00000000006e1);
```

### `int::bit::reverse_bytes(n)`

Reverses the [byte order](https://en.wikipedia.org/wiki/Endianness) of the int `n`.

```pavo
assert(int::bit::reverse_bytes(0x1234567890123456), 0x5634129078563412);
```

### `int::bit::reverse_bits(n)`

Reverses the binary representation of the int `n`.

```pavo
assert(int::bit::reverse_bits(0x1234567890123456), 0x6a2c48091e6a2c48);
```

### `int::bit::shl(n, m)`

Performs a [logical left shift](https://en.wikipedia.org/wiki/Logical_shift) of the int `n` by the positive int `m` many bits. This always results in `0` if `m` is greater than `63`.

```pavo
assert(int::bit::shl(5, 1), 10);
assert(int::bit::shl(42, 64), 0);
```

### `int::bit::shr(n, m)`

Performs a [logical right shift](https://en.wikipedia.org/wiki/Logical_shift) of the int `n` by the int `m` many bits. This always results in `0` if `m` is greater than `63`.

```pavo
assert(int::bit::shr(5, 1), 2);
assert(int::bit::shr(42, 64), 0);
```

### `array`

This module bundles functions operating on arrays.

### `arr::count(arr)`

Returns the number of elements in the array `arr`.

Time: O(1).

```pavo
assert(arr::count([]), 0);
assert(arr::count([nil]), 1);
assert(arr::count([0, 1, 2]), 3);
```

### `arr::get(arr, i)`

Returns the element at the index `i` in the array `arr`.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::get([true, false], 0), true);
assert(arr::get([true, false], 1), false);
```

### `arr::insert(arr, p, new)`

Inserts the value `new` into the array `arr` at the position `p`.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::insert([0, 1], 0, 42), [42, 0, 1]);
assert(arr::insert([0, 1], 1, 42), [0, 42, 1]);
assert(arr::insert([0, 1], 2, 42), [0, 1, 42]);
```

### `arr::remove(arr, i)`

Returns the array obtained by removing the element at the index `i` from the array `arr`.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::remove([0, 1], 0), [1]);
assert(arr::remove([0, 1], 1), [0]);
```

### `arr::set(arr, i, v)`

Returns the array obtained by replacing the element at the index `i` in the array `arr` with the value `v`.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::set([0, 1], 0, 42), [42, 1]);
assert(arr::set([0, 1], 1, 42), [0, 42]);
```

### `arr::split(arr, p)`

Splits the array `arr` at the position `p`, returning an array containing two arrays: The first containing the subsequence of the elements between positions `0` to `p`, the second containing the subsequence consisting of the remaining elements.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::split([0, 1, 2], 0), [[], [0, 1, 2]]);
assert(arr::split([0, 1, 2], 1), [[0], [1, 2]]);
assert(arr::split([0, 1, 2], 2), [[0 1], [2]]);
assert(arr::split([0, 1, 2], 3), [[0 1 2], []]);
```

### `arr::slice(arr, start, end)`

Returns an array containing the subsequence of the elements of the array `arr` between position `start` and position `end`.

The vm aborts if `start` is strictly greater than `end`.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::slice([true, false], 1, 1), []);
assert(arr::slice([true, false], 0, 1), [true]);
assert(arr::slice([true, false], 1, 2), [false]);
assert(arr::slice([true, false], 0, 2), [true, false]);
```

### `arr::slice_to(arr, p)`

Returns an array containing the subsequence of the elements of the array `arr` from the beginning to the position `p`.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::slice_to([true, false], 0), []);
assert(arr::slice_to([true, false], 1), [true]);
assert(arr::slice_to([true, false], 2), [true, false]);
```

### `arr::slice_from(arr, p)`

Returns an array containing the subsequence of the elements of the array `arr` from the position `p` to the end.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::slice_from([true, false], 0), [true, false]);
assert(arr::slice_from([true, false], 1), [true]);
assert(arr::slice_from([true, false], 2), []);
```

### `arr::splice(old, p, new)`

Inserts the elements of the array `new` into the array `old`, starting at the index position `p`.

Time: O(log (n + m)), where n is `arr::count(old)` and m is `arr::count(new)`.

```pavo
assert(arr::splice([0, 1], 0, [10, 11]), [10, 11, 0, 1]);
assert(arr::splice([0, 1], 1, [10, 11]), [0, 10, 11, 1]);
assert(arr::splice([0, 1], 2, [10, 11]), [0, 1, 10, 11]);
```

### `arr::concat(left, right)`

Returns an array that contains all elements of the array `left` followed by all elements of the array `right`.

Time: O(log (n + m)), where n is `arr::count(left)` and m is `arr::count(right)`.

```pavo
assert(arr::concat([0, 1], [2, 3]), [0, 1, 2, 3]);
assert(arr::concat([], [0, 1]), [0, 1]);
assert(arr::concat([0, 1], []), [0, 1]);
```

### `arr::builder()`

Returns a new array builder, which is an object that allows efficiently constructing arrays. The internal state consists of an ordered buffer, from which an array can be efficiently created.

#### `arr_builder("push_start", [v])`

Adds the value `v` to the start of the internal buffer of the builder.

Time: amortized O(n), where n is the number of values that have already been pushed into this builder.

#### `arr_builder("push_end", [v])`

Adds the value `v` to the end of the internal buffer of the builder.

Time: amortized O(n), where n is the number of values that have already been pushed into this builder.

#### `arr_builder("build", [])`

Returns an array containing the values that have been added to the internal buffer.

```pavo
let b = arr::builder();

b("push_start", [0]);
b("push_end", [1]);
b("push_start", [2]);
b("push_end", [3]);

assert(b("build", []), [2, 0, 1, 3]);
```

### `arr::focus(arr, p)`

Returns a new array focus, which is an object that allows efficiently accessing values inside an array that are close to each other. The internal state consists of the elements of the array `arr`, and a designated position within that array, initially `p`. The focus provides methods to efficiently access elements close to the current position, and for efficiently changing that position to another, nearby position.

#### `arr_focus("refocus", [by])`

Updates the internal position by adding the int `by` to it, saturating at the boundaries for valid positions, i.e. zero and the length of the focussed array. Returns by how much the position was changed (may be negative).

Time: amortized O(by). Traversing the complete array in steps of size `by` must take at most O(c) time, where c is the number of items in the focused array.

```pavo
let f = arr::focus([0, 1, 2, 3, 4], 4); # [0 1 2 3 * 4]
assert(f("refocus", [-1]), -1); # [0 1 2 3 * 4];
assert(f("refocus", [-2]), -2); # [0 1 * 2 3 4];
assert(f("refocus", [9]), 3); # [0 1 2 3 4 *];
```

#### `arr_focus("get_relative", [i])`

Computes an index by adding the int `i` to the current position of the focus. If this is a valid index in the focussed array, returns `{"ok": v}`, where `v` is the value at that position. Otherwise, returns `{"err": nil}`.

Time: amortized O(i). Traversing the complete array and calling this method on every position must take at most O(c * i) time, where c is the number of items in the focused array.

```pavo
let f = arr::focus([0, 1, 2, 3, 4], 4); # [0 1 2 3 * 4]
assert(f("get_relative", [0]), {"ok": 4});
assert(f("get_relative", [-1]), {"ok": 3});
assert(f("get_relative", [-9]), {"err": nil});
```

### `arr::check`

Functions operating on arrays that check whether indices or positions are out of bounds, and return results accordingly.

### `arr::check::get(arr, i)`

Returns the element at the positive int `i` in the array `arr`. Returns `{"ok": v}` if `v` is the value at index `i`, or `{"err": nil}` if `i` is not a valid index.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::check::get([true, false], 0), {"ok": true});
assert(arr::check::get([true, false], 1), {"ok": false});
assert(arr::check::get([true, false], 2), {"err": nil});
```

### `arr::check::insert(arr, p, new)`

Inserts the value `new` into the array `arr` at the position `p`. Returns `{"ok": new}` if `new` is the resulting array, or `{"err": nil}` if `p` is not a valid position.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::check::insert([0, 1], 0, 42), {"ok": [42, 0, 1]});
assert(arr::check::insert([0, 1], 1, 42), {"ok": [0, 42, 1]});
assert(arr::check::insert([0, 1], 2, 42), {"ok": [0, 1, 42]});
assert(arr::check::insert([0, 1], 3, 42), {"err": nil});
```

### `arr::check::remove(arr, i)`

Returns the array obtained by removing the element at the index `i` from the array `arr`. Returns `{"ok": new}` if `new` is the resulting array, or `{"err": nil}` if `i` is not a valid index.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::check::remove([0, 1], 0), {"ok": [1]});
assert(arr::check::remove([0, 1], 1), {"ok": [0]});
assert(arr::check::remove([0, 1], 2), {"err": nil});
```

### `arr::check::set(arr, i, v)`

Returns the array obtained by replacing the element at the index `i` in the array `arr` with the value `v`. Returns `{"ok": new}` if `new` is the resulting array, or `{"err": nil}` if `i` is not a valid index.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::check::set([0, 1], 0, 42), {"ok": [42, 1]});
assert(arr::check::set([0, 1], 1, 42), {"ok": [0, 42]});
assert(arr::check::set([0, 1], 2, 42), {"err": nil});
```

### `arr::check::split(arr, p)`

Splits the array `arr` at the position `p`, returning `{"ok": new}`, where `new` is an array containing two arrays: The first containing the subsequence of the elements between positions `0` to `p`, the second containing the subsequence consisting of the remaining elements. Returns `{"err": nil}` if `p` is not a valid position.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::check::split([0, 1, 2], 0), {"ok": [[], [0, 1, 2]]});
assert(arr::check::split([0, 1, 2], 1), {"ok": [[0], [1, 2]]});
assert(arr::check::split([0, 1, 2], 2), {"ok": [[0 1], [2]]});
assert(arr::check::split([0, 1, 2], 3), {"ok": [[0 1 2], []]});
assert(arr::check::split([0, 1, 2], 4), {"err": nil});
```

### `arr::check::slice(arr, start, end)`

Returns an array containing the subsequence of the elements of the array `arr` between position `start` and position `end`. Returns `{"ok": new}` if `new` is the resulting array, or `{"err": nil}` if `start` is not a valid position, `end` is not a valid position, or `start` is strictly greater than `end`.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::check::slice([true, false], 1, 1), {"ok": []});
assert(arr::check::slice([true, false], 0, 1), {"ok": [true]});
assert(arr::check::slice([true, false], 1, 2), {"ok": [false]});
assert(arr::check::slice([true, false], 0, 2), {"ok": [true, false]});
assert(arr::check::slice([true, false], 3, 4), {"err": nil});
assert(arr::check::slice([true, false], 0, 3), {"err": nil});
assert(arr::check::slice([true, false], 2, 1), {"err": nil});
```

### `arr::check::slice_to(arr, p)`

Returns an array containing the subsequence of the elements of the array `arr` from the beginning to the position `p`.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::check::slice_to([true, false], 0), {"ok": []});
assert(arr::check::slice_to([true, false], 1), {"ok": [true]});
assert(arr::check::slice_to([true, false], 2), {"ok": [true, false]});
```

### `arr::check::slice_from(arr, p)`

Returns an array containing the subsequence of the elements of the array `arr` from the position `p` to the end.

Time: O(log n), where n is `arr::count(arr)`.

```pavo
assert(arr::check::slice_from([true, false], 0), {"ok": [true, false]});
assert(arr::check::slice_from([true, false], 1), {"ok": [true]});
assert(arr::check::slice_from([true, false], 2), {"ok": []});
```

### `arr::check::splice(old, p, new)`

Inserts the elements of the array `new` into the array `old`, starting at the index position `p`. Returns `{"ok": arr}` if `arr` is the resulting array, or `{"err": nil}` if `p` is not a valid position.

Time: O(log (n + m)), where n is `arr::count(old)` and m is `arr::count(new)`.

```pavo
assert(arr::check::splice([0, 1], 0, [10, 11]), {"ok": [10, 11, 0, 1]});
assert(arr::check::splice([0, 1], 1, [10, 11]), {"ok": [0, 10, 11, 1]});
assert(arr::check::splice([0, 1], 2, [10, 11]), {"ok": [0, 1, 10, 11]});
assert(arr::check::splice([0, 1], 3, [10, 11]), {"err": nil});
```











TODO

- map
- function
- preemptive_yield
- buffers

# Valuable Value Virtual Machine

An instantiation of the [generic unityped virtual machine](https://github.com/AljoschaMeyer/guvm) (guvm), using a superset of the [valuable values](https://github.com/AljoschaMeyer/valuable-value). All mutability stems from mutable closures, collections are immutable.

The deterministic version of the vvvm uses the deterministic guvm, no further nondeterminism is introduced.

## Values

All [valuable values](https://github.com/AljoschaMeyer/valuable-value) are vvvm values, as are the functions provided by a guvm. Functions can be (transitively) contained in composite valuable values (arrays and maps).

The rest of this document lists all the built-in (asynchronous) functions provided by the vvvm. They are grouped into categories which are also recommended name spaces for programming languages running on the vvvm.

## Built-In Functions

### `value`

This module bundles functions which operate on values of any type.

### `value::halt(v)`

Abnormally halts program execution without producing a value. The vvvm should present the argument to the outside world for diagnostic purposes.

### `value::type_of(v)`

Returns a string describing what kind of value the argument is: `"nil"`, `"bool"`, `"float"`, `"int"`, `"array"`, `"map"`, or `"function"`.

```pavo
assert(value::type_of(nil), "nil");
assert(value::type_of(false), "bool");
assert(value::type_of(0.1), "float");
assert(value::type_of(42), "int");
assert(value::type_of([]), "array");
assert(value::type_of({}), "map");
assert(value::type_of(value::type_of), "function");
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

### `value::total`

This module bundles functions for interacting with the total order over all values. This order is a straightforward extension of the total order over all valuable values: functions are greater than maps, built-in functions are ordered in the order in which they appear in this document, closures are greater than built-in functions, and are ordered by ordinal.

See `value::total::compare` for examples illustrating how the order works.

### `value::total::compare(v, w)`

Returns `"<"` if `v` is strictly less than `w` in the total order over all values, `"="` if they are equal, and `">"` if `v` is strictly greater than `w`.

```pavo
assert(value::total::compare(0, 1), "<");
assert(value::total::compare(1, 1), "=");
assert(value::total::compare(1, 0), ">");

# `nil` is less than any other value
assert(value::total::compare(nil, false), "<");

# `false` is less than `true`
assert(value::total::compare(false, true), "<");

# bools are less than floats
assert(value::total::compare(true, 0.1), "<");

# floats are ordered numerically, with `NaN` being the greatest and negative
# zero being less than positive zero
assert(value::total::compare(-Inf, -999.0), "<");
assert(value::total::compare(-999.0, -0.0), "<");
assert(value::total::compare(-0.0, 0.0), "<");
assert(value::total::compare(0.0, 999.0), "<");
assert(value::total::compare(999.0, Inf), "<");
assert(value::total::compare(Inf, NaN), "<");

# floats are less than ints
assert(value::total::compare(1.0, 0), "<");

# ints are ordered numerically
assert(value::total::compare(-2, 1), "<");
assert(value::total::compare(-0, 0), "=");

# ints are less than arrays
assert(value::total::compare(99, []), "<");

# arrays are ordered lexicographically
assert(value::total::compare([], [1, 1]), "<");
assert(value::total::compare([0, 9, 9], [1, 1]), "<");
assert(value::total::compare([1], [1, 1]), "<");
assert(value::total::compare([1, 0], [1, 1]), "<");
assert(value::total::compare([1, 1], [1, 1, 0]), "<");
assert(value::total::compare([1, 1], [2]), "<");
assert(value::total::compare([1, 1], [2, 0]), "<");

# arrays are less than maps
assert(value::total::compare([999], {}), "<");

# maps are ordered as if they were arrays containing their entries as
# two-element arrays, the outer array being sorted by keys:
# for example `{1: "b", 0: "a"}` corresponds to `[[0, "a"], [1, "b"]]`
assert(value::total::compare({}, {0: 90, 1: 80}), "<");
assert(value::total::compare({0: 89, 1: 99, 2: 99}, {0: 90, 1: 80}), "<");
assert(value::total::compare({0: 90, 1: 79}, {0: 90, 1: 80}), "<");
assert(value::total::compare({0: 90, 1: 80}, {0: 90, 1: 80, 2: 0}), "<");
assert(value::total::compare({0: 90, 1: 80}, {1: 79}), "<");

# maps are less than functions
assert(value::total::compare({}, value::type_of), "<");

# built-in functions are ordered by their position in this document
assert(value::total::compare(value::type_of, value::truthy), "<");
assert(value::total::compare(value::truthy, value::truthy), "=");

# built-in functions are less than closures
assert(value::total::compare(value::type_of, (x) -> { return x;}), "<");

# closures are ordered by their ordinal, which isn't necessarily deterministic
# in deterministic mode, creation time determines the ordinal:
assert(value::total::compare((x) -> { return x;}, (x) -> { return x;}), "<");
```

### `value::total::lt(v, w)`

Returns `true` if `v` is strictly less than `w` in the total order over all values, `false` otherwise.

```pavo
assert(value::total::lt(0, 1), true);
assert(value::total::lt(1, 1), false);
assert(value::total::lt(1, 0), false);
```

### `value::total::leq(v, w)`

Returns `true` if `v` is less than or equal to `w` in the total order over all values, `false` otherwise.

```pavo
assert(value::total::leq(0, 1), true);
assert(value::total::leq(1, 1), true);
assert(value::total::leq(1, 0), false);
```

### `value::total::eq(v, w)`

Returns `true` if `v` is equal to `w` in the total order over all values, `false` otherwise.

```pavo
assert(value::total::eq(0, 1), false);
assert(value::total::eq(1, 1), true);
assert(value::total::eq(1, 0), false);
```

### `value::total::geq(v, w)`

Returns `true` if `v` is greater than or equal to `w` in the total order over all values, `false` otherwise.

```pavo
assert(value::total::geq(0, 1), false);
assert(value::total::geq(1, 1), true);
assert(value::total::geq(1, 0), true);
```

### `value::total::gt(v, w)`

Returns `true` if `v` is strictly greater than `w` in the total order over all values, `false` otherwise.

```pavo
assert(value::total::gt(0, 1), false);
assert(value::total::gt(1, 1), false);
assert(value::total::gt(1, 0), true);
```

### `value::total::neq(v, w)`

Returns `true` if `v` is not equal to `w` in the total order over all values, `false` otherwise.

```pavo
assert(value::total::neq(0, 1), true);
assert(value::total::neq(1, 1), false);
assert(value::total::neq(1, 0), true);
```

### `value::total::min(v, w)`

Returns the argument that is less in the total order over all values.

```pavo
assert(value::total::min(0, 1), 0);
assert(value::total::min(1, 1), 1);
assert(value::total::min(1, 0), 0);
```

### `value::total::max(v, w)`

Returns the argument that is greater in the total order over all values.

```pavo
assert(value::total::max(0, 1), 1);
assert(value::total::max(1, 1), 1);
assert(value::total::max(1, 0), 1);
```

### `value::partial`

This module bundles functions for interacting with the meaningful partial order over all values. This order is a straightforward extension of the meaningful partial order over all valuable values: functions are incomparable to other kinds of values, two functions are compared as of the total order over all values.

See `value::partial::compare` for examples illustrating how the order works.

### `value::partial::compare(v, w)`

Returns `{"ok": "<"}` if `v` is strictly less than `w` in the meaningful partial order over all values, `{"ok": "="}` if they are equal, and `{"ok": ">"}` if `v` is strictly greater than `w`. Returns `{"err": nil}` if `v` and `w` are incomparable.

```pavo
assert(value::partial::compare(0, 1), {"ok": "<"});
assert(value::partial::compare(1, 1), {"ok": "="});
assert(value::partial::compare(1, 0), {"ok": ">"});

# different kinds of values are incomparable
assert(value::partial::compare(42, []), `{"err": nil}`);

# `false` is less than `true`
assert(value::partial::compare(false, true), {"ok": "<"});

# floats are ordered numerically, with `NaN` being the greatest and negative
# zero being less than positive zero
assert(value::partial::compare(-Inf, -999.0), {"ok": "<"});
assert(value::partial::compare(-999.0, -0.0), "<"{"ok": "<"});
assert(value::partial::compare(-0.0, 0.0), {"ok": "<"});
assert(value::partial::compare(0.0, 999.0), {"ok": "<"});
assert(value::partial::compare(999.0, Inf), {"ok": "<"});
assert(value::partial::compare(Inf, NaN), {"ok": "<"});

# ints are ordered numerically
assert(value::partial::compare(-2, 1), {"ok": "<"});
assert(value::partial::compare(-0, 0), {"ok": "="});

# arrays are ordered component-wise
assert(value::partial::compare([], [1, 1]), {"ok": "<"});
assert(value::partial::compare([1, 0], [1, 1]), {"ok": "<"});
assert(value::partial::compare([0, 1], [1, 1]), {"ok": "<"});
assert(value::partial::compare([0, 2], [1, 1]), {"err": nil});
assert(value::partial::compare([2, 0], [1, 1]), {"err": nil});
assert(value::partial::compare([0, 1, 0], [1, 1]), `{"err": nil}`);
assert(value::partial::compare([2, 1], [1, 1]), {"ok": ">"});
assert(value::partial::compare([1, 2], [1, 1]), {"ok": ">"});
assert(value::partial::compare([1, 1, 0], [1, 1]), {"ok": ">"});

# maps are ordered entry-wise
assert(value::partial::compare({}, {0: 90, 1: 80}), {"ok": "<"});
assert(value::partial::compare({0: 90, 1: 79}, {0: 90, 1: 80}), "<");
assert(value::partial::compare({0: 90, 1: 80}, {0: 90, 1: 80, 2: 0}), "<");
assert(value::partial::compare({0: 89, 1: 81}, {0: 90, 1: 80}), `{"err": nil}`);
assert(value::partial::compare({0: 5}, {0: 4, 1: 0}), `{"err": nil}`);

# built-in functions are ordered by their position in this document
assert(value::partial::compare(value::type_of, value::truthy), {"ok": "<"});
assert(value::partial::compare(value::truthy, value::truthy), {"ok": "="});

# built-in functions are less than closures
assert(value::partial::compare(value::type_of, (x) -> { return x;}), {"ok": "<"});

# closures are ordered by their ordinal, which isn't necessarily deterministic
# in deterministic mode, creation time determines the ordinal:
assert(value::partial::compare((x) -> { return x;}, (x) -> { return x;}), {"ok": "<"});
```

### `value::partial::lt(v, w)`

Returns `{"ok": true}` if `v` is strictly less than `w` in the meaningful partial order over all values, `{"ok": false}` if `v` is greater than or equal to `w`, `{"err": nil}` if they are incomparable.

```pavo
assert(value::partial::lt(0, 1), {"ok": true});
assert(value::partial::lt(1, 1), {"ok": false});
assert(value::partial::lt(1, 0), {"ok": false});
assert(value::partial::lt(1, true), {"err": nil});
```

### `value::partial::leq(v, w)`

Returns `{"ok": true}` if `v` is less than or equal to `w` in the meaningful partial order over all values, `{"ok": false}` if `v` is strictly greater than `w`, `{"err": nil}` if they are incomparable.

```pavo
assert(value::partial::leq(0, 1), {"ok": true});
assert(value::partial::leq(1, 1), {"ok": true});
assert(value::partial::leq(1, 0), {"ok": false});
assert(value::partial::leq(1, true), {"err": nil});
```

### `value::partial::eq(v, w)`

Returns `{"ok": true}` if `v` is equal to `w` in the meaningful partial order over all values, `{"ok": false}` if one of them strictly greater than the other, `{"err": nil}` if they are incomparable.

```pavo
assert(value::partial::eq(0, 1), {"ok": false});
assert(value::partial::eq(1, 1), {"ok": true});
assert(value::partial::eq(1, 0), {"ok": false});
assert(value::partial::eq(1, true), {"err": nil});
```

### `value::partial::geq(v, w)`

Returns `{"ok": true}` if `v` is greater than or equal to `w` in the meaningful partial order over all values, `{"ok": false}` if `v` is strictly less than `w`, `{"err": nil}` if they are incomparable.

```pavo
assert(value::partial::geq(0, 1), {"ok": false});
assert(value::partial::geq(1, 1), {"ok": true});
assert(value::partial::geq(1, 0), {"ok": true});
assert(value::partial::geq(1, true), {"err": nil});
```

### `value::partial::gt(v, w)`

Returns `{"ok": true}` if `v` is strictly greater than `w` in the meaningful partial order over all values, `{"ok": false}` if `v` is less than or equal to `w`, `{"err": nil}` if they are incomparable.

```pavo
assert(value::partial::gt(0, 1), {"ok": false});
assert(value::partial::gt(1, 1), {"ok": false});
assert(value::partial::gt(1, 0), {"ok": true});
assert(value::partial::gt(1, true), {"err": nil});
```

### `value::partial::neq(v, w)`

Returns `{"ok": true}` if `v` is not equal to `w` in the meaningful partial order over all values, `{"ok": false}` if `v` is equal to `w`, `{"err": nil}` they are incomparable.

```pavo
assert(value::partial::neq(0, 1), {"ok": true});
assert(value::partial::neq(1, 1), {"ok": false});
assert(value::partial::neq(1, 0), {"ok": true});
assert(value::partial::neq(1, true), {"err": nil});
```

### `value::partial::greatest_lower_bound(v, w)`

Returns `{"ok": glb}`, where `glb` is the greatest value less than or equal to both arguments, or {"err": nil} if no such value exists.

```pavo
assert(value::partial::greatest_lower_bound({0: 90}, {0: 89, 1: 80}), {"ok": {0: 89}});
assert(value::partial::greatest_lower_bound(1, true), {"err": nil});
```

### `value::partial::least_upper_bound(v, w)`

Returns `{"ok": gub}`, where `gub` is the least value greater than or equal to both arguments, or {"err": nil} if no such value exists.

```pavo
assert(value::partial::least_upper_bound({0: 90}, {0: 89, 1: 80}), {"ok": {0: 90, 1: 80}});
assert(value::partial::least_upper_bound(1, true), {"err": nil});
```












TODO

- bool
- float
- int
- array
- map
- function
- preemptive_yield
- builders, iterators

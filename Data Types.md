## Scalar Types

### Integer Types

| Length | Signed | Unsigned |
| ------ | ------ | -------- |
| 8-bit  | i8     | u8       |
| 16-bit | i16    | u16      |
| 32-bit | i32    | u32      |
| 64-bit | i64    | u64      |
| arch   | isize  | usize    |
>ℹ The `isize` and `usize` types depend on the kind of computer your program is running on: 64 bits if you're on a 64-bit architecture and 32 bit if you're on a 32-bit architecture.


| Number literals | Example     |
| --------------- | ----------- |
| Decimal         | 98_765      |
| Hex             | 0xff        |
| Octal           | 0o77        |
| Binary          | 0b1111_0000 |
| Byte (u8 only)  | b'A'        |
## Compound Types

## Tuple type

A tuple is a general way of grouping together some number of other values with a variety of types into one compound type.

```rust
let tup: (i32, f64, u8) = (500, 6.4, 1);
// destructuring a tuple
let (x, y, y) = tup;
// accessing tuple elements by index
let five_hundred = tup.0;
let six_point_four = tup.1;
let one = tup.2;
```

## Array type

Every element of an array must have the same type, and fixed length: once declared, they cannot grow or shrink in size.

```rust
let array = [1, 2, 3, 4, 5];
// accessing array elements
let first = a[0];
let second = a[1];
```
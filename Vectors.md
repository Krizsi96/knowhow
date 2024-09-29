
Vectors are a collection type (`Vec<T>`) that allow you to store more than one value in a single data structure that puts all the values next to each other in memory. Vector can only store values of the same type.

## Creating a new Vector

```rust
// creating a new vector
let v: Vec<i32> = Vec::new();
// creating a vector from values
let v = vec![1, 2, 3];
// creating a vector from an array
let arr = [1, 2, 3, 4, 5];
let v: Vec<i32> = arr.into();
```

### Updating a Vector

```rust
let mut v = Vec::new();

v.push(5);
v.push(6);
```
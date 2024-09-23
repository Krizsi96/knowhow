
## Associated function

An associated function is implemented on a type, rather than on a particular instance of the type. Some languages call this a `static method`.#

## Cargo

It is the build system and package manager of Rust. Cargo handles a lot of tasks for you, such as building your code, downloading the libraries your code depends on, and building those libraries. (more info on [cargo](https://doc.rust-lang.org/cargo/))

create a new Rust project:

```shell
cargo new hello_cargo --bin
```

`--bin` argument makes an executable application as opposed to a library.

build the project:

```shell
cargo build --release
```

`--release` argument builds the release version of the software instead of the debug version.

run the project:

```shell
cargo run
```

checking if the code compiles, but not creating an executable:

```shell
cargo check
```

## Closure

Sometimes it is useful to wrap up a function and _free variables_ for better clarity and reuse. The free variables that can be used come from the enclosing scope and are ‘closed over’ when used in the function. From this, we get the name ‘closures’.
Rust’s implementation of closures is a bit different than other languages. They are effectively syntax sugar for [[Rust Lexicon#Traits]].

Closures look like this:
``` Rust
let plus_one = |x: i32| x + 1;

assert_eq!(2, plust_one(1));
```

We create a binding, `plus_onej`, and assign it to a closure. The closer's arguments go between the pipes (`|`), and the body is an expression. In this case, `x + 1`. 

Remember that `{ }` is an expression, so we can have multi-line closures too:

``` Rust
let plus_two = |x| {
	let mut result: i32 = x;

	result += 1;
	result += 1;

	result
};

assert_eq!(4, plus_two(2));
```

### Taking closures as arguments

We can write a function which takes something callable, calls it, and returns the result:

``` Rust
fn call_with_one<F>(some_closure: F) -> i32
	where F: Fn(i32) -> i32 {
		
	some_closure(1)
}

let answer = call_with_one(|x| x + 2);

assert_eq!(3, answer);
```

We pass our closure, `|x| x + 2`, to `call_with_one`. It does what it suggest: it calls the closure, giving it `1` as an argument.

```
where F: Fn(i32) -> i32
```

Because `Fn` is a trait, we can use it as a bound for our generic type. In this case, our closure takes an `i32` as an argument and returns an `i32`, and so the generic bound we us is `Fn(i32) -> i32`.

Of course, if we want dynamic dispatch, we can get that too. A trait object handles this case, as usual:
``` Rust
fn call_with_one(some_closure: &Fn(i32) -> i32) -> i32 {     some_closure(1) }  let answer = call_with_one(&|x| x + 2);  assert_eq!(3, answer);
```

Now we take a trait object, a `&Fn`. And we have to make a reference to our closure when we pass it to `call_with_one`, so we use `&||`.

## Dependency Management

The packages in rust are called `crates`. You can search for the available open source crates on [crates.io](https://crates.io/).

## Panic

In Rust, for normal recoverable errors you would just return a result type in the error state from whatever function you're in. Panic is reserved for fatal errors, it's when you've hit some intractable issue like failing an assertion and the `panic handler` is where your code goes to die. 💀

## Prelude

## rustc

The Rust C compiler is a front end for [[LLVM]].

### rustc/LLVM target triple

`<arch><sub>-<vendor>-<sys>-<env>`

- arch ▶ core architecture (x86_64, i386, arm, ...)
- sub ▶ sub architecture (ex. arm v5, v6, v7m, ...)
- vendor (pc, apple, ibm, ...)
- sys ▶ operating system (none, linux, win32, darwin, ...)
- env ▶ environment or ABI (eabi, gnu, elf, ...)

## Traits


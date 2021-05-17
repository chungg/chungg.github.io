---
layout: post
title:  "python to rust: the basics"
date:   2021-05-17 16:00:00 -0500
tags: rust python programming
---

The following shows Rust code and equivalent code in Python. Disclaimer: the following is not
guaranteed to work as is (in fact most likely won't).

variables
=========

```rust
// foo = 'a'
let foo: char = 'a';

// to allow variable to be modified
let mut foo: i8 = 1;

// const must be initialised where as regular variables can be declared at first (but not common)
// can't be a function and can be global
const FOOBAR: u32 = 1;

// stating the obvious, but Rust is a statically typed language so we need to set a type when
// declaring a variable. In some cases, it's fine to not provide type and compiler will choose
// based on initial value but not necessarily optimally.
let integer = 1;

// can set variables without explicit function. below set y to whatever x is
let y = {
    let mut x = 0;
    // code
    x // note missing semi-colon
};
```

data types
==========

tuples
******
```rust
// tup = (1, 2 '2')
let tup = (1u8, 2u16, '2');

// indexing
// tup[1]
tup.1;

// slicing
// tup[:2]
// from what i can tell, tuples aren't meant for slicing/iterating
// more like an namedtuple in python (without the name)
```

array
*****
```rust
// i don't use array lib in python
// arr = numpy.arange(1, 6, dtype=numpy.int32)
let arr: [i32; 5] = [1, 2, 3, 4, 5];

// arr = numpy.zeros(500, dtype=numpy.int32)
let arr: [u32; 500] = [0; 500];

// indexing
// arr[0]
arr[0];

// slicing
// arr[1:4]
&arr[1..4];

// len(arr)
arr.len();
```

std lib types
*************

string
------
In Rust there are `String` and `&str`. The latter is most aligned with python as in python
strings are immutable.

```rust
// value = "the quick brown fox"
let value = "the quick brown fox";

let mut string = String::new();
for c in chars {
    // Insert a char at the end of string
    string.push(c);
    // Insert a string at the end of string
    string.push_str(", ");
}
```

vector/list
------
vectors appear to be similar to lists in python as in their size is dynamic
```rust
// values = list(range(10))
// values = [1, 2, 3]
let values: Vec<i32> = (0..10).collect();
let mut values = vec![1, 2, 3];

// values.append(4)
// values.pop()
values.push(4);
values.pop();

// values[2]
// len(values)
values[2];
values.len();

// for i in values:
//    print(i)
for i in values.iter() {
    println!("{:?}", i);
}
// into_iter() will effectively throw away values from vec as it iterates
// iter_mut() allows code to modify the `i`

```

hashmap
-------
```rust
// hm = dict()
use std::collections::HashMap;
let mut hm = HashMap::new();

// hm['foo'] = 'bar'
// hm['foo']
hm.insert("foo".to_string(),  "bar".to_string());
hm["foo"]
// 'value' in hm
hm.contains_key("value")
// if 'value' not in hm:
//     hm['value'] = 'hi'
hm.entry("value").or_insert("hi".to_string())
```

set
---
```rust
// values = set()
use std::collections::HashSet;
let mut values = HashSet::new();

// values.add('blah')
values.insert("blah".to_string())
// 'foo' in values
values.contains("foo")
```

casting
*******
```rust
// blah = 32.1
// blah = int(blah)
let blah = 32.1;
let blah = blah as u16;

// chr(32)
let int_as_char = 32 as char;

// casting to lower type will cropped to max
let blah = 300.0_f32 as u8;  // will return 255
```

flow
====

if/else
*******
```rust
// if value == 1:
//     print("hi")
// elif value == 2:
//     print("bye")
// else:
//     print("leave now")
if value == 1 {
    println!("hi");
} else if value == 2 {
    println!("bye");
} else {
    println!("leave now");
}
```

switch/case/match
*****************
```rust
// only exist in python3.10
match value {
    1 => println!("hi"),
    2 => println!("bye"),
    x if x > 0 => println!("can add guard for additional conditional"),
    x @ 3 .. 10 => println!("can bind with @ to get access to value {}", x),
    _ => println!("leave now"),
}
```

loops
*****

for
---
```rust
// for i in range(1, 100):
//     print(i)
for i in 1..100 {
    println!("{}", i);
}
```

while
-----
```rust
// while True:
//     print("hi")
//     break
loop {
    println!("hi");
    break;
}

// x = 0
// while x < 5:
//     print(x)
//     x += 1
let mut x = 0
while x < 5 {
    println!("{}", x);
    x += 1;
}
```

iterating
---------
```rust
// any(i for i in arr if i == 2)
arr.iter().any(|&x| x == 2)
// value = None
// for i in arr:
//     if i == 2:
//         value = i
//         break
value = arr.iter().find(|&x| *x == 2)
// pos = None
// for count, i enumerate(arr):
//     if i == 2:
//         pos = count
//         break
value = arr.iter().position(|&x| x == 2)
```

more
----
```rust
// break multiple loops
'outer: loop {
    println!("Entered the outer loop");

    'inner: loop {
        println!("Entered the inner loop");

        // This breaks the outer loop as well
        break 'outer;
    }

// return a value. below sets result = "value"
let result = loop {
    // something
    break "value";
}
```

functions
=========
```rust
// def do_something(foo: int, bar: int) -> bool
//     return False
// value = 1
// do_something(value, 2)
fn do_something(foo: i32, bar: i32) -> bool
    return false; // or false
do_something(&value, 2) // see ownership for `&`

```

lambdas/closures
****************
```rust
// something = lambda x: x
let something = |x: i32| -> i32> { x };
// something = lambda x: x.append(1)
// add move to modify the input parameter
let something = move |x: Vec<i32>| -> () { x.push(1); };
```

"classes"
=========
```rust
// class Something:
//     x: int
//     y: int
//     def does_nothing(self):
//         pass
struct Something {
    x: i64,
    y: i64,
}

impl Something {
    fn does_nothing(&self) -> () {
        // if self not included, it is a static method
        // &mut self to modify object variables. object itself must be mutable as well.
    }    
}
```

scope
=====
```rust
let x = 1;
{
    println!("Can see x's value: {}" , x);
    let y = 2;
}
println!("Cannot see y here");

{
    let x = 3;
    // x is 3 here
}
println!("x is still 1");
// not affected by variable shadowing in previous block
```

- `self::<func|struct>` to call item in same module
- `super::<func|struct>` to call item in parent scope
- `crate::<func|struct>` to call item in crate scope
- `pub <use|fn|mod|struct>` to all access outside current module

modules
*******

import/use
**********
```rust
// from a.b.c import d, e
// from a.b.c import *
// from a.b.c import d as newname
use a::b::c::{d, e};
use a::b::c::*;
use a::b::c::d as newname;
```

ownership
=========
```rust
// only one variable can own a value
let s1 = String::from("asdf");
let s2 = s1;
// s1 no longer has ownership and can't access string
// doesn't apply to primitives like int/float/bool which will create a copy
let int1 = 1;
let int2 = int2;
// int1 and int2 are both valid variables at this point

// only one variable can borrow a value
let s3 = &mut s2;
let s4 = &s2;
// let s5 = &mut s2 will fail
```

debugging
=========

```rust
// print(f'the value of x is {x}')
// if non-primitive type, neeed to implement Display
// impl fmt::Display for CustomStruct {
//     fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
//         write!(f, "{}", self.0)
//     }
// }
println!("the value of x is {}, x);


// debugger printing
// closer to print(f'the value of x is {repr(x)}')
// if x is a non-primitive type, need to 'decorate' the struct with `#[derive(Debug)]` to print
println!("the value of x is {:?}", x);
```

TODO: add details on debugger (if it exists)

packaging
=========
```bash
// poetry new my-package
cargo new my-package 
```

---
layout: post
title: Rust's great error handling capability.
date: 09-04-2023
categories: programming
hero: https://res.cloudinary.com/dbzsk4ytb/image/upload/v1681039714/blog-images/ErrorHandling_z10t3w.jpg
attrib: "(c) fizkes/Adobe Stock - licensed for use"
---

Handling errors is hard. Doing it well is one of the key differences between great developers and average ones. Every developer desires solid, robust code that does its job and doesn’t crash – and is easy to maintain. The language designers of Rust have provided unique means of defining and handling errors fundamentally different from the programming languages that preceded it – and is a key contributor to why developers have continuously voted Rust the “most loved language” on Stack Overflow’s yearly developer survey. This article will examine the differences between Rust language’s error-handling mechanisms and prior programming languages and how Rust’s approach is better.

## In the beginning

The C programming language was introduced in the 1970s and became the first widely popular coding language. But its error-handling mechanisms are rudimentary. The following is very typical and idiomatic C code for reading from a file:

``` c
void a_test() {
  FILE *fp = fopen("filename", "r");
  char buff[255];

  while(fread(buff, 1, 255, fp)) {
  // do something with buff
  }
}
```

What's surprising about this code (at least to non-C developers) is that `fread()` does not return a `bool` value – it returns a value of type `size_t` (or `int`), which is the number of bytes read. The rules of C state that the integer value 0 is treated as false; anything else is true. So our loop keeps going as long as the `fread()` function returns some bytes. When a zero is returned, either an error happened or we've reached the end of the file.

This pattern of intermixing valid return values with error codes is pervasive in C and in most other programming languages where a function can only return a single value. C has a global ERRNO value that may have more information about what went wrong, but in general, you don't get a lot of extra data about what kind of error occurred. When you can only return one value from a function and want the function to return useful data, you must mix error codes with the return values. This can cause problems – the return value must be something that is not in the set of valid values. We see -1 and 0 used as “magical error return values” in many functions to indicate an error has occurred. 

## Structured Error Handling

To do better, C++ (and later, Java) introduced structured error handling – the try/catch/throw mechanism that allowed errors to be triggered in one place, have additional data attached to the error, and differentiate between different kinds of errors. 

``` cpp
#include <iostream>

int call_a_function() {
    std::cout << "inside of my function" << std::endl;
    throw("some error");
    std::cout << "we will not get this far" << std::endl;
    return 42;
}

int main(int argc, const char * argv[]) {
    int my_answer;
    std::cout << "Hello, World!" << std::endl;
    try {
        my_answer = call_a_function();
    } catch (const char* s) {
        std::cout << "An error has occurred: " << s << std::endl;
        my_answer = -1;
    }
    std::cout << "My answer is: " << my_answer << std::endl;
    return 0;
}

Output:
Hello, World!
inside of my function
An error has occurred: some error
My answer is: -1
```

While exception handling did allow for cleanly separating errors from valid return values, this approach has fundamental problems. People have written lengthy arguments about the issues with structured error handling ([Exception Handling Considered Harmful](https://www.lighterra.com/papers/exceptionsharmful/), for example). The main drawback with exception handling is that it introduces an alternate flow path for a program that is hard to see as a developer, much less control. When the exception is thrown, we do not know who will catch it – it could be several functions up the call stack. Also, a function catching an exception might be surprised by who threw the exception – we cannot choose which functions we want to catch exceptions from – anything that runs underneath our called function is possible. 

Exception handling also introduces problems in multi-threaded applications – separating exception handling across threads is difficult. Java’s requirement that any function that could throw an exception list the exceptions in its method definition is also repetitive and verbose.

Structured error handling is not meant for “normal” error conditions – it is meant for errors that disrupt an application’s normal flow, not routine return values from a function. A string find operation that returns the position of a character inside of a string wouldn’t throw an exception if it can’t find the intended character – it would instead return some “not found” result. The C function `strchr()` does this – it returns either a pointer to the character found or the magical value `null`, indicating failure. You wouldn’t expect this function to throw an exception if it can’t find a character. Exception handling only attempts to solve part of the problem of returning error conditions out of functions.

## A Different approach

Go took a novel approach – it allows functions to return multiple values instead of one. You will frequently see the following in Golang code: 

``` go
return_value, err = my_func();
if err != nil {
	return err;
}
```

Here both the error code and the value you need are returned from a function – they do not try to occupy the same spot. But, the error value placeholder is present when an error does not occur. It's up to the programmer to check the error value, and `if err != nil` statements are littered throughout Go code. But sometimes you don't want to deal with the error – there's nothing useful to do – and you want to return it from your function. There is a significant amount of boilerplate error-handling code in Go programs to check for errors on each function call in an application, as there is no other way to handle the error.

## Rust's approach

Rust has a novel approach to error handling that leverages Rust’s enumerations. Enumerations in Rust are algebraic data types – they support data values, not just constants. Two special-purpose enumerations are commonly used to handle return values: `Option` and `Result`. `Option` is used when a function may or may not return a useful value – you get a `Some(value)` or `None` as your values. Using our find a character in a string example, the Rust function `find` returns an `Option` enumeration – it will have a value of `Some(x)` with `x` being the character’s position or a value of `None`. You can use Rust’s pattern-matching mechanism to handle these conditions elegantly:

``` rust
let a_string = "testing".to_string();
let my_result = a_string.find('a');

match my_result {
    Some(pos) => println!("I found the character at position: {}", pos),
    None => println!("I did not find the character")
}
```

Here we have a precise mechanism for knowing if the function returned a valid value or did not work. You do not have to check for a magical “error value”; the enumeration allows for the result to be separate from the error condition.
 
Rust's pattern-matching rules require that your code handle both cases of the enumeration – you can't silently skip error handling as the compiler enforces it. But Rust has a `?` operator that allows you to stop and return `None` if that's what comes back from a function. The following 2 code snippets are identical in function:

``` rust
fn string_test() -> Option<usize> {
    let a_string = "testing".to_string();
    if let Some(pos) = a_string.find('a') {
        println!("I found the character at position: {}", pos);
        Some(pos)
    } else {
        None
    }
}
```

And

``` rust
fn string_test() -> Option<usize> {
    let a_string = "testing".to_string();
    let pos = a_string.find('a')?;
    println!("I found the character at position: {}", pos);

    Some(pos)
}
```

The `?` operator “unwraps” the `Option`, pulling the value out or returning from the calling function the value `None` if it is not there. Note that we are not bypassing the error handling – it is still occurring, and the Rust compiler does it for us automatically. This is very clean and does not clutter the code – making it easier to follow the flow of the function.

The second enumeration used in error handling is `Result`. The two possible values of Result are `Ok(T)` and `Err(E)`. Here we are returning an error value, and not a plain `None` so that we can pass more information back to the calling function when there is an error. Here is an example function that is reading a file, where it returns an error back to the calling function if the file does not exist or there is some other type of error reading the file:

``` rust
fn get_data(filename: &str) -> Result<String, Error> {
    let mut my_file = File::open(filename)?;
    let mut buffer = String::new();

    let bytes_read = my_file.read_to_string(&mut buffer)?;
    Ok(format!("read {} bytes, string is:\n{}", bytes_read, buffer))
}
```

Here, our function returns a `Result`, which will contain either a `String` value or an error of type `Error`. We again use the `?` operator on the `read_to_string()` function, which unwraps the number of bytes read on a successful return; otherwise, it returns an `Err` to the calling function. We print out the number of bytes read and the content of our data file on a successful read. A simple `main()` that calls this function could look like this:

``` rust
fn main() {
    match get_data("data.txt") {
        Ok(my_string) => println!("result: <{}>", my_string),
        Err(e) => eprintln!("error of: {}", e),
    };
}
```

When we run our program and have a valid data.txt file, we can see the following output:

```
% cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/rust_examples`
result: <read 65 bytes, string is:
This is some data in a text file.
Here is a second line of data.
>
```

When the data file does not exist, here is what our output looks like:

```
% cargo run
    Finished dev [unoptimized + debuginfo] target(s) in 0.00s
     Running `target/debug/rust_examples`
error of: No such file or directory (os error 2)
```

We see that a complete error was propagated back to our main function, where we captured it in our match statement and could print it out to the console.

Our simple examples demonstrate how Rust’s error handling enables cleaner code, where an application can elegantly handle the various conditions that calling an external function might produce. The `?` operator with the `Option` and `Result` enumerations allow for a very clean means of testing for errors, handling them appropriately, and not relying on mixing valid values and error codes. An ability to attach meaningful data to error conditions makes for more robust APIs.

There is more – Rust’s standard libraries have a host of specific functions for managing error states and mapping library errors into user-defined values. External libraries like the `thiserror` crate extend Rust's standard error handling. The `thiserror` crate provides an easy-to-use means of defining specific errors that look and feel exactly like the built-in error types that Rust defines: (example derived from `thiserror` on `docs.rs`)

``` rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum DataStoreError {
    #[error("data store disconnected")]
    Disconnect(#[from] io::Error),
    #[error("the data for key `{0}` is not available")]
    Redaction(String),
    #[error("invalid header (expected {expected:?}, found {found:?})")]
    InvalidHeader {
        expected: String,
        found: String,
    },
    #[error("unknown data store error")]
    Unknown,
}
```

The `anyhow` crate defines a flexible means of handling multiple error types, and adding application-specific data to errors. Anyhow allows for `Result<T, anyhow::Error>`, or equivalently `anyhow::Result<T>`, as the return type of a function. A function can then return any error type that implements the `std::error::Error` trait. (example derived from `anyhow` on `docs.rs`)

``` rust
use anyhow::Result;

fn get_cluster_info() -> Result<ClusterMap> {
    // plain return of an error, here likely related to file system errors
    let config = std::fs::read_to_string("cluster.json")?;
    
    // return of an error with context
    let map: ClusterMap = serde_json::from_str(&config)
        .context(“json deserialization failed, file cluster.json likely not formatted correctly”)?;
    
    Ok(map)
}
```

Our simple examples show how Rust’s error handling significantly differs from prior common programming languages. Rust provides a clean means of separating good vs. bad results from functions without introducing boilerplate that litters our source code. And we have the tools needed to define new error types that work cleanly within our library or application. 

Designing appropriate error handling is still a hard job – but Rust at least gives us the tools to do a good job of it! And this is more than aesthetics – better error handling allows errors to be more visible in an application and decreases application defects due to not handling error conditions correctly. It also contributes to cleaner and simpler designs – by definition, better designs – which make applications more robust and easier to develop correctly. This is a solid contributor to the Rust programming language’s high interest and “love” from developers!
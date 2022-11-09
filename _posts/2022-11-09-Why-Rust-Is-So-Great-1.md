---
layout: post
title: Why Rust is so Great – Reason 1, The Borrow Checker
date: 09-11-2022
categories: programming
hero: https://res.cloudinary.com/dbzsk4ytb/image/upload/c_scale,w_720/v1667997352/blog-images/Rust-is-Great/Great_rgfa03.jpg
---
When I first tripped across Rust, it was through an article touting Rust's election as the "most loved programming language" for several years in Stack overflow's yearly survey (https://survey.stackoverflow.co/2022/#technology-most-loved-dreaded-and-wanted). The high-level language description was interesting – compiled, no virtual machine, efficient, fast, and safe. I'll admit a bias towards compiled and type-safe languages – I grew up on C, have taught C, C++, and Java, and have used Java for most of my professional career. Very early in my research, I ran across a bunch of articles of the type "Why Rust is better than <insert programming language here>" and "Why you should learn Rust." They all seem to have come from the same root source – and mostly cover points that any statically typed language that doesn't run on a virtual machine runtime would have. "Blazingly fast" "catches variable type mismatch at compile time" are examples.

What is missing is an explanation of what is unique to Rust that other languages do not have. I want to tackle the first reason in this article – the Rust Borrow Checker.

The inventors of Rust benefit from years of practical experience with other programming languages. They have designed a set of features in the language and associated tooling that address many of the pain points facing application developers. First up is memory management – from the beginning, programs have dealt with how to use memory safely, and the Rust Borrow Checker is a novel approach to solving this problem.

## Some history

First, let me review some history of how programming languages have tried to tackle this problem. In the beginning, there was C, and the low-level and often-cursed "malloc()" and "free()." C is a programming language that gives developers complete control of their environment – it is a small step up from assembly language. Developers get to fully manage memory access for their application and how to give allocated memory back to the operating system. This low-level control takes a lot of careful planning. But, it was the source of many runtime errors, through either memory leaks (not letting go of memory) or crashes (freeing the same pointer twice or using memory after it is released are common problems). It is also a source of security issues – reading previously freed memory for data you're not supposed to have is a possible exploit. Here's an example program that demonstrates what the compiler allows but is incorrect:

### C Code
```cpp
#include<stdio.h>
#include<stdlib.h>

int *get_me_some_memory(int how_much) {
    return (int *)malloc(sizeof(int) * how_much);
}

int main(){
  int *my_memory = get_me_some_memory(25);
  
  my_memory[5] = 12;
  my_memory[30] = 15; // Note, access beyond the end of my allocation

  free(my_memory);

  int bad_value = my_memory[5]; // Note, access to memory after a free  

  printf("%d\n", bad_value);
  return 0;
}

* Output of the above when compiled and executed, MacOs
> cc c-malloc-example.c
> ./a.out
> 12

* Same source code, compiled with compiler optimizations turned on:
> cc -Ofast c-malloc-example.c
> ./a.out
> 0
```

C++ enabled better control over memory allocation – classes had constructors that would allocate memory for their objects, and destructors freed that memory. Smart Pointers also were created that helped with automatically freeing memory when variables fell out of scope. But the "foot guns" of low-level pointers and references made errors challenging to eliminate. You were ok if you followed the rules – but if you weren't aware of the rules, the compiler wasn't of any help. The hidden complexity inside objects also made it difficult to follow the rules and predict how your application would perform.

## Garbage Collection

Programming languages turned to garbage collection as a means to solve these problems. Garbage collection-based programs can grab whatever memory they need from a runtime environment. In the background, the garbage collection process will release memory back to the operating system once the program is no longer using it. Scripting languages that rely on interpreting source code at run time, including Basic, Python, and JavaScript, use this approach in the run time environment of their interpreters. Java was one of the first compiled languages that targeted running within a runtime (the JVM) that performed garbage collection. New programming languages, including Golang (Go) and Dart, utilize Garbage Collection. These environments are productive for developers – it makes managing memory much simpler. Garbage collectors have gotten very sophisticated, and many application types work well in this environment.

But Garbage Collection imposes limitations. The overhead imposed by the Garbage Collector impacts the application's runtime behavior. Pauses in the application need to be scheduled, which makes Garbage Collected languages unsuitable for real-time programming. The runtime that goes along with the environment is also typically heavyweight, which gets in the way when running in the small environments of embedded and IoT devices – or containerized microservices. Here's a quick example java application that is similar to our C version that shows how garbage collection is useful:

### Java Code
```java
public class java_allocation_example {
    public static void main(String[] args) { 
        
          int []my_memory = get_me_some_memory(25);
          
          my_memory[5] = 12;  // Completely normal and valid
          my_memory[30] = 15; // Note, access beyond the end of my allocation, this blows up in Java
        
          int []my_new_memory = my_memory; // I have a new variable - does this get a copy, or does it just
                                           // point to the original?

          System.out.println("Initial value: " + my_memory[5]); // Still 12, as it should be

          my_new_memory[5] = 7; // using my 2nd variable to update the array

          System.out.println("New value: " + my_new_memory[5]); // This prints out "7", my new value
          System.out.println("Original array: " + my_memory[5]); // What does this print???
          
          my_memory = null; // Doesn't free anything, but the GC could now reclaim our memory if
                            // it wanted to.
                            
          //Will cause a null pointer exception.
          System.out.println("Use after null: " + my_memory[5]); 
          
          return;
        }
        
    private static int[] get_me_some_memory(int size) {
        return new int[size]; 
    }   
}

* Terminal execution:
> java java_allocation_example.java
Exception in thread "main" java.lang.ArrayIndexOutOfBoundsException: Index 30 out of bounds for length 25
        at java_allocation_example.main(java_allocation_example.java:7)

* after all bad lines are commented out:        
> java java_allocation_example.java
Initial value: 12
New value: 7
Original array: 7

```

Everything compiles, but we get consistent crashes at run time instead of undefined runtime behavior – that's good and bad. Predictable behavior is good, but the java "NullPointerException" runtime error has crashed applications in production more times than anyone would want to admit. When we comment out the bad line, we now see a side effect – the new variable did not receive a completely new version of the array; it just "pointed" to the original. When we modified the new variable, the original one had its contents modified out from under it. In this trivial example, we can follow what is happening here – but if that modification occurred deep within a nested library function, we could easily be surprised.

But, our memory management is more straightforward – we get to request the memory space we need at run time and don't have to deal with letting it go. The ease of use is why garbage collection is widespread, and mainstream languages JavaScript, Python, Go, and Dart are using it today.

## What Rust does differently

Rust's approach to memory management enables the ease of memory management of a Garbage Collection environment *but without the runtime environment overhead or associated performance penalty*. It also solves the unexpected side effect problems we saw in our Java example. A smart pointer-style mechanism is a key – but instead of relying on programmers to follow usage rules (like C++), the compiler enforces access rules. "Fighting the borrow checker" is one of the first things new Rust developers figure out how to do – but once mastered, it enables developers to manage their memory usage automatically.

Along with a productive development environment, Rust's borrow checker eliminates a whole class of programming errors related to memory management. Surveys have demonstrated that many of the security vulnerabilities present in applications are due to a program's inability to protect access to allocated memory. Rust programs run fast, enable developers to be efficient, are more secure, and reduce runtime crashes. Let's look at a Rust application that mimics our examples:

### Finally - some Rust code!
```rust
fn get_me_some_memory(how_much: usize) -> Vec<usize> {
  vec![0; how_much]   //Rust fun - can skip the "return" when skipping the trailing ;
}

fn main() {
    let mut some_vector;
    let mut my_vector = get_me_some_memory(25);
    let mut my_array: [usize; 50] = [0; 50];

    my_vector[5] = 12; // Completely normal and valid
    my_array[55] = 17; // Going off the deep end of the array - this blows up at
                       // compile time, not run time.
    my_vector[30] = 15; // A dynamic vector makes it through a compile, but panics at
                        // run time.

    some_vector = my_vector; //typical assigning to a new variable - or is it?

    let value = some_vector[5]; // Look the vector moved to its new home, everything works
    println!("{}\n", value);

    some_vector[5] = 9; // I can re-assign my value just fine.

    let old_value = my_vector[5]; // This doesn't compile - my_vector is no longer valid here
    println!("{}\n", old_value);
    return;
}
```

We again have a function that allocates a block of memory on the heap using Rust's Vector type. A Vector is a contiguous block of memory that is dynamically sized, accessed like an array, and stored on the heap. By contrast, line 8 allocates an array, which is fixed in size at compile time and allocates on the stack. Since we know the array size at compile time, we get a compiler warning on line 12 – Rust knows that we're going past the end of the array.

### Rust's compiler keeps us safe
```rust
> cargo build
   Compiling rust_examples v0.1.0 
error: this operation will panic at runtime
  --> src\main.rs:11:5
   |
12 |     my_array[55] = 17; // Going off the end of the array - this blows up 
   |     ^^^^^^^^^^^^ index out of bounds: the length is 50 but the index is 55
   |
   = note: `#[deny(unconditional_panic)]` on by default

error: could not compile `rust_examples` due to previous error


* Commenting out the array (lines 8 and 12), we now get another error:
> cargo build
   Compiling rust_examples v0.1.0 
error[E0382]: borrow of moved value: `my_vector`
  --> src\main.rs:23:21
   |
7  |     let mut my_vector = get_me_some_memory(25);
   |         ------------- move occurs because `my_vector` has type `Vec<usize>`, which does not implement the `Copy` trait
...
16 |     some_vector = my_vector; //typical assigning to a new variable - or is it?
   |                   --------- value moved here
...
23 |     let old_value = my_vector[5]; // This doesn't compile - my_memory is no longer valid here
   |                     ^^^^^^^^^ value borrowed here after move

For more information about this error, try `rustc --explain E0382`.
error: could not compile `rust_examples` due to previous error
```

The rust compiler is excellent at giving us meaningful error information. On line 16, the vector is "moved" to a new home – it no longer exists at my_vector. Rust keeps track of the allocated memory but ensures that only one variable "owns" the memory and can make changes. 

We can assign a variable to our vector if we state it is an immutable variable. We can only "read" the values through this new variable, not write to them. Rust enforces a rule that there can only be 1 "writer" / "owner" at a time – that way, it prevents all "whoever saves last wins" errors, the unexpected side effect issues we saw in our Java example. It can also keep track of the variables using the allocated memory to ensure it gets freed when all of the variables using the memory fall out of scope. Here's our example program with an immutable variable pointing to our vector:

### Updated listing, showing borrowing with an immutable variable
```rust
fn main() {
    let mut some_vector;
    let mut my_vector = get_me_some_memory(25);
    //let mut my_array: [usize; 50] = [0; 50];
    let my_read_only_pointer; //notice no "mut" here, we cannot write values
                              //using this variable, only read them

    my_vector[5] = 12; // Completely normal and valid
    //my_array[55] = 17; // Going off the deep end of the array - this blows up at
                       // compile time
    my_vector[30] = 15; // A dynamic vector makes it through compile, but panics at
                        // run time

    my_read_only_pointer = &my_vector; // Borrowing access to the vector
    let abc = my_read_only_pointer[5];
    println!("{}", abc);

    some_vector = my_vector; //typical assigning to a new variable - or is it?

    let value = some_vector[5]; // Look the vector moved to its new home, everything works
    println!("{}\n", value);

    some_vector[5] = 9; // I can re-assign my value just fine.
    
    let old_value = my_vector[5]; // This doesn't compile - my_memory is no longer valid here
    println!("{}\n", old_value);
    return;
}
```

## The magic of Rust's borrow checker is this:

1.	Rust ensures ownership of any memory (heap or stack) is in one place. Only the owner of the data can manipulate it. Multiple read-only access is allowed.
2.	Rust keeps track of the memory variables used and automatically frees up memory when all the variables using that memory are out of scope – whether on the stack (our array example) or the heap (our Vector example). 
3.	Bounds checking is enforced either at compile time (for arrays, which have their sizes determined at compile time) or run time (vectors, which have their sizes set at run time). 
4.	These rules eliminate a whole class of errors and security vulnerabilities. We can't write to memory after a free() – Rust keeps memory around for us as long as anything has access. These rules are all checked at compile time. We don't have to worry about letting go of memory – Rust takes care of that. Boundary checks also prevent us from writing outside of the memory space allocated to us.
5.	Most of this happens at compile time – we do not have a runtime garbage collector that has to run alongside us, and there are no interruptions to the program execution to allow a garbage collector to interrogate memory usage. A common Rust saying is: "if it compiles, it will work."

Even better, how the borrow checker works enable multiple threads to use allocated memory safely. When programs move into multiple threads of execution, the opportunity for error expands exponentially. "Fearless concurrency" – using data safely across multiple threads automatically – is a significant benefit of Rust's borrow checker.

Rust's memory management and the borrow checker are two main reasons Rust is different. You can access memory at a low level with complete control, without a garbage collector runtime or the risks of a runtime error blowing up your program unexpectedly. This feature makes Rust great for low-level systems, embedded, or real-time applications where efficiency is necessary. It also enables Rust to be used for higher-level applications such as web service publishing, as the "memory automatically frees" environment is productive for developers.

But that's not all – follow me to be notified when I publish the next article on another feature of Rust that makes it great!

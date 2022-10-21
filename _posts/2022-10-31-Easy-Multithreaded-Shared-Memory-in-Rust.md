---
layout: post
title: Easy multi-threaded shared memory in rust
date: 31-10-2022
categories: programming
hero: https://res.cloudinary.com/dbzsk4ytb/image/upload/c_scale,w_720/v1666362943/blog-images/AdobeStock_93027465_qcp9os.jpg
---

This article is the 3rd in a series related to building a wireless thermostat in Rust, running on a Raspberry Pi – although this one has very little to do with the Raspberry Pi and is more relevant to any multi-threaded application. When creating my application, I implemented a simple and effective design for controlling multi-threaded access to shared memory. You can find the previous articles here: [first article - Raspberry Pi Wireless Thermostat in Rust](https://mhentges.com/rpi-thermostat) and [second article - Cross Compiling Made Easy](https://mhentges.com/rust-cross-compiling-made-easy). I have a few code snippets below, but you can find the entire code repository [on github](https://github.com/mikehentges/thermostat-pi).

## First, the back story:

When designing a multi-threaded application, one of the core considerations is how the different threads share data. There's always something that the threads share – you're spinning off threads to work on a shared problem, after all. If nothing else, configuration data or database connection pools have to be shared. There are two main ways of approaching this problem.
1.	**Message Passing** – channels between threads allow data to be sent and received between them. Message passing is the mechanism that golang prefers, as the golang documentation states: "Do not communicate by sharing memory; instead, share memory by communicating." Rust supports creating channels; you can find more information in the Rust Book's chapter on [message passing]( https://doc.rust-lang.org/book/ch16-02-message-passing.html).
2.	**Shared-State** – a set of data identified as a shared state. Each thread access a shared data area through thread-safe protection mechanisms – typically a Mutex. Locks on the data allow you to access the data safely, reading or updating the data as needed, and then the lock is released. Only one thread can read or write the data at a time. The Rust Book's [shared state](https://doc.rust-lang.org/book/ch16-03-shared-state.html) section describes these mechanisms.

Both approaches work, and Rust has good support for both mechanisms in the standard library. To choose between them, consider the following. 

1.	**What are the access patterns?** Will threads mainly read data, mostly write data, or both?
2.	**How will deadlock be avoided?** Avoiding deadlock is a primary concern for multi-threaded applications – nothing is worse than having a bunch of threads available to do work, and they are all stuck waiting for the other ones to get out of the way. 
3.	**Performance.** While over-engineering for performance is a common fault, having some idea of the performance requirements of your solution is essential. I'm a big fan of the rule: *keep the solution as simple as possible, but no simpler*.

I decided on a shared state approach for the application I am creating. This decision also implied that my worker threads would work by polling – checking on things periodically instead of receiving an outside message. Here is what led me to that conclusion.

1.	I have three main threads. One is a web interface that allows for reading/writing of the shared state (get temperature, get thermostat setting, set thermostat setting). The web interface is, by definition, a creator of unscheduled activity – things come in based on an external client's action and are not under the application's control. Second, a pair of background worker threads react to the application's environment. The first reads the temperature sensor, and the second calculates whether the thermostat should be set on or off and controls the physical relay to make that happen. The thermostat is in a separate thread, as it could turn on/off when the temperature crosses a threshold, or we receive a new thermostat setting from the web interface. 
2.	I didn't have to worry about contention with only three threads (assuming a single external client on the web interface). A simple shared state approach wouldn't run into issues based on too many threads trying to access locks simultaneously.
3.	The logic for determining the thermostat's state has a time component. We don't want frequent, short bursts of on/off, which thrash the furnace. We enforce a minimum amount of time that the thermostat will be on or off before switching to another state. Accurately reacting to a message (the temperature reading or thermostat value changes) requires knowing how long it's been since the thermostat changed.

A shared state made sense based on these requirements. My design enforces a low thread count, so contention was not an issue (a common problem with shared state approaches). I need to keep track of time, so some state data was always required. Implementing two different state data mechanisms in the same application would be added complexity. With each thread doing its own thing and getting/setting shared state independently, we isolate each thread from the others. This choice led to a simpler overall application design.

But, as I reviewed the Rust Book's content on [shared-state](https://doc.rust-lang.org/book/ch16-03-shared-state.html),
the complexity of managing access from each thread seemed daunting. Plus, sprinkling thread-locking code in the application made it a mess – my nice single-purpose functions now had Mutex locking logic. Also, when accessing data through a Mutex's lock dealing with Rust's borrow checker across threads proved challenging.

## A simpler approach:

To make this manageable, I used an approach I've used before on C++ projects – encapsulating the shared data into a separate class and pushing all Mutex logic into the class. Rust doesn't have classes, but a couple of structs did the trick.

To start, I created a struct to hold all the shared state data in a single place. Then I could deal with a single reference to this struct instead of managing each data element independently.

```rust
pub struct SharedData {
    continue_background_tasks: bool,
    current_temp: f32,
    thermostat_value: usize,
    thermostat_on: bool,
    thermostat_change_datetime: OffsetDateTime,
}
```

Then we define a struct that holds an Arc pointer – an Atomic Reference Count pointer to a Mutex that guards our shared data struct. We use this struct to control access to/from our shared data. 

```rust
pub struct AccessSharedData {
    pub sd: Arc<Mutex<SharedData>>,
}
```

We will make many copies of this pointer – they all point to our Mutex, which is how we gain access to our shared memory space. We do this by customizing the Clone() method for AccessSharedData – it looks like the following.

```rust
// Clone here makes a copy of the Arc pointer - not  the entire class of data
// All clones point to the same internal data
impl Clone for AccessSharedData {
    fn clone(&self) -> Self {
        AccessSharedData {
            sd: Arc::clone(&self.sd),
        }
    }
}
```

To use this, we first create an instance of our SharedData struct (the simple .new() method isn't shown above but is straightforward).

```rust
let common_data = SharedData::new(
    true,
    configuration.initial_thermostat_value as f32 + 5.0,
    configuration.initial_thermostat_value,
    false,
    OffsetDateTime::UNIX_EPOCH,
);
```
Then, we initialize an instance of AccessSharedData.

```rust
// The wrapper around our shared data that gives it safe access across threads
let sd = AccessSharedData {
    sd: Arc::new(Mutex::new(common_data)),
};
```

We then give each thread we spawn() a cloned copy of the AccessSharedData struct. The clone call creates a copy of the Arc pointer, which we then move into the new thread. A similar method passes in a clone of the AccessSharedData struct to the actix_web HttpServer::new() method, so it is also available in HTTP client handlers.

```rust
// Create another clone of our pointer to shared data, and send it into a new thread that continuously
// checks to see how the current temperature and current thermostat setting compare - and will
// trigger turning on the relay for the furnace as needed.
let sdc = sd.clone();
let control_handle = spawn(async move {
  tracing::debug!("kicking off control_thermostat");
  match run_control_thermostat(&sdc, configuration.poll_interval).await {
    Ok(_) => tracing::info!("control_thermostat ended"),
    Err(e) => tracing::error!("control_thermostat returned an error {:?}", e),
  }
});
```

Now that everyone has a copy of the Arc pointer, we need to create a means to use it to read/write our shared data struct. A simple set of getters/setters for each member of our struct handles the task of acquiring a lock, getting/setting the data, and automatically releasing the lock. By putting this logic surrounding the get/set and utilizing the end of the method as a scope boundary to force the lock to be released, we gain control over the access to our data.

```rust
impl AccessSharedData {
    pub fn continue_background_tasks(&self) -> bool {
        let lock = self.sd.lock().unwrap();
        lock.continue_background_tasks
    }
    pub fn set_continue_background_tasks(&self, new_val: bool) {
        let mut lock = self.sd.lock().unwrap();
        lock.continue_background_tasks = new_val;
    }
    //repeated for remaining struct members
}
```

When each get/set function returns, the lock is released. This approach also means it is impossible to deadlock around access to the structure – everything is locked and released immediately and prohibits anything else from getting in the way. With all of our locking mechanisms in place, using the shared data in the rest of our application is trivial.

```rust
sd.set_current_temp(temp_f);
let temp_now = sd.current_temp();
let thermostat_now = sd.thermostat_value();
```

## In Summary

This strategy is encapsulation at its finest. Application code uses straightforward calls to methods to get/set the shared data elements whenever needed. Behind the scenes, we manage the Mutex's locking for that access to be thread-safe and safe from deadlocks. And Rust's borrow checker is extremely happy that we aren't allowing direct shared mutable references to our shared data. If needed, you could easily extend this design pattern to implement a more robust access pattern.

Putting locking logic inside getters/setters enables our shared data struct to hide the complexities of multi-threaded access – much like the standard library does for thread-safe collections. Using clones of an Arc makes it possible to acquire a handle to the shared memory struct in each thread. These two concepts are the core ideas that make our Easy Shared Data design pattern work.

 
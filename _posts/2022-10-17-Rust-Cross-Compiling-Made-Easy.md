---
layout: post
title: Rust Cross Compiling Made Easy
date: 17-10-2022
categories: programming
hero: https://res.cloudinary.com/dbzsk4ytb/image/upload/c_scale,w_720/v1666029626/blog-images/louis-reed-zDxlNcdUzxk-unsplash_awz9h9.jpg
---

This article is the second installment of my series on building a wireless thermostat in Rust for the Raspberry Pi. You can find the [beginning of the series here](https://mhentges.com/rpi-thermostat). All source code for the project is [located here](https://github.com/mikehentges/thermostat-pi). 

As I approached the task of building a native Rust executable for the Raspberry Pi, one of the first things I had to tackle was establishing a cross-compiling development environment. The Raspberry Pi runs a flavor of Unix, but we need to compile executables for the Pi's ARM processor. 

My first attempt followed the process documented in the Cross-compilation chapter of [The Rustup Book](https://rust-lang.github.io/rustup/cross-compilation.html). I had to find the appropriate target for a Raspberry Pi Zero (my target): ```arm-unknown-linux-gnueabihf```. This target will be slightly different if you want to target a Pi 3 or Pi 4 – then you can use ```target=armv7-unknown-linux-gnueabihf```. The Pi Zero runs a v6 version of the ARM processor, larger Pi's use a v7 (or higher) version. 

Cross-compiling sounded easy, and following the Rustup Book's directions added cross-compiling to my environment:

```
rustup target add arm-unknown-linux-gnueabihf
cargo build –target= arm-unknown-linux-gnueabihf
```

But that doesn't work out of the box. Even the documentation warns you: *"Note that rustup target add only installs the Rust standard library for a given target. There are typically other tools necessary to cross-compile, particularly a linker."* Then, my application used SSL – and I also needed a way to cross-compile the openssl library for the Pi. I build under WSL2 (Ubuntu) on a Windows machine – getting this set up would take some effort.

Fortunately, there's an easier way! The [Rust Embedded Devices Tools Team](https://github.com/rust-embedded/wg#the-tools-team) publishes a set of Docker images that contain a complete toolchain for a large number of cross-compile targets. Even better, they created a " cross " tool that automates the entire process of launching a suitable Docker container, getting your code attached to the container, and running the compile process. Incremental builds are fully supported, so cross-compiling does not take much longer than native compiling. 

To set this up, you first to install the "cross" tool:

```
cargo install cross
```

Then, compiling any project for a new target is as simple as substituting "cross" for "cargo" in your command. I also had to pass a features flag to get the OpenSSL library built from source for the target platform:

```
cross build --release –target=arm-unknown-linux-gnueabihf --features vendored-openssl
```

That command will build a release executable at ```./target/arm-unknown-linux-gnueabihf/release/thermostat-pi``` (my project is “thermostat-pi”). 

For another part of the project, I needed to cross-compile to x86_64-unknown-linux-musl to create an AWS Lambda using Rust. The [AWS Rust SDK](https://docs.aws.amazon.com/sdk-for-rust/latest/dg/lambda.html) documents the steps and includes the "Container approach" that uses cross:

```
cross build --release --target x86_64-unknown-linux-musl
```

You can find the lambda project in the ./push_temp folder in the (source repository)[https://github.com/mikehentges/thermostat-pi/tree/main/push-temp].

All these build commands are too long for me to remember. Command-line history helps a ton, but returning to the project after an extended break can mean looking up the build commands again. Instead, I recently found [just](https://github.com/casey/just), a lighter-weight version of make that is purpose-built as a command runner. You can create a justfile, using syntax similar to make's Makefile, to define a set of commands. Dependencies are supported, allowing you to bundle together different commands as needed. For the ./push_temp project, its very simple justfile looks like this:

```makefile
build: 
	cross build --release --target x86_64-unknown-linux-musl
	cp target/x86_64-unknown-linux-musl/release/push_temp bootstrap
test:
	curl -X POST -F "record_date=2022:02:03T15:50:00" -F "thermostat_on=true" -F "temperature=55" -F "thermostat_value=60" https://5zvz7wehuh.execute-api.us-east-2.amazonaws.com/test_lambda
```

This setup allows a simple ```just build`` command – which I can remember! The test target sends a sample transaction to my lambda function – again, keeping in a central place a longer command that I can reuse as needed. 

I also created a justfile to run the cross-compile of the main project and automated the deployment of the executable to the Raspberry Pi. The justfile I created looks like this:

```makefile
TARGET_HOST := "mhentges@shop-therm.local"
TARGET_PATH := "/home/mhentges/thermostat-pi"
TARGET_ARCH := "arm-unknown-linux-gnueabihf"
SOURCE_PATH := "./target/" + TARGET_ARCH + "/release/thermostat-pi"

check:
    cargo check

build:
    cross build --release --target={{TARGET_ARCH}} --features vendored-openssl
    rsync ./configuration.yaml {{TARGET_HOST}}:/home/mhentges/configuration.yaml
    rsync {{SOURCE_PATH}} {{TARGET_HOST}}:{{TARGET_PATH}}
    #ssh -t {{TARGET_HOST}} {{TARGET_PATH}}

clippy:
    cargo clippy
```

This script relies on SSH being available between your host machine and the PI, configured with [public key authentication](https://www.digitalocean.com/community/tutorials/how-to-configure-ssh-key-based-authentication-on-a-linux-server).  

Using the Cross tools is the secret sauce that makes cross-compiling for Rust easy. Whether you use Windows, macOS, Linux, or rent a dev box in the cloud, you can set up the cross-compiling environment with a few simple commands. Then, use just to provide automation to allow for easy-to-execute build and deploy steps.  

  <p align="center" style="font-size:small">hero image by <a href="https://unsplash.com/@_louisreed?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Louis Reed</a> on <a href="https://unsplash.com/s/photos/raspberry-pi?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText">Unsplash</a>
</p>

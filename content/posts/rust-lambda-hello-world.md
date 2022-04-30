---
title: "Rust Lambda Hello World"
date: 2022-03-07T17:06:12-05:00
draft: false
description: A quick tutorial to create a "hello world" lambda in Rust, using the AWS Lambda Rust Runtime
---


## Getting started with AWS Lambda Rust Runtime

I just finished a quick tutorial to create a [hello world](https://github.com/millerjam/rust_lambda_hello_world) lambda using the [AWS Lambda Rust Runtime](https://github.com/awslabs/aws-lambda-rust-runtime)

---
#### Github repo for the tutorial
 If you're in a rush...

 You can view the completed source code here: [rust_lambda_hello_world](https://github.com/millerjam/rust_lambda_hello_world)

---
## Overview

I think that Rust is a great fit for serverless and I wanted to start documenting how to use Rust in a lambda. Lambdas are a great opportunity to test out a new language without making a huge commitment in terms of time or resources. Additionally, the explicit connection between execution time and cost highlight the benefit of Rust's speed, both for for cold starts and  warm execution time.

Before getting started here some prerequisites that you should already have working...

## Prerequisites
1. Install rust/cargo/etc
1. Code editor
1. AWS Account

## Create a new rust binary
Let's get started with creating a new rust binary

```
cargo new lambda_test
```
Here I am using `cargo` to create a new binary with the name "lambda_test". This will create a new directory and create a scaffold of a rust project.

## Getting started

Now we are ready to get started writing our first lambda in rust. We will create a simple "hello world" lambda that will read an event that contains a first name, and respond with "Hello [first name]." Along the way we will cover writing the code, adding the crates, cross compiling for the lambda environment, deploying, and finally testing our code.

Let's get started...

## Add the code

To make this easy, we'll copy the code from the [AWS Lambda Rust Runtime](https://github.com/awslabs/aws-lambda-rust-runtime/tree/v0.5.0) README example. (Note: make sure you've selected the tag for version, v0.5 at the time I'm writing this) Copy the code snippet below, and replace the existing code in your project's `lambda_test/main.rs` file.

```rust
use lambda_runtime::{service_fn, LambdaEvent, Error};
use serde_json::{json, Value};

#[tokio::main]
async fn main() -> Result<(), Error> {
    let func = service_fn(func);
    lambda_runtime::run(func).await?;
    Ok(())
}

async fn func(event: LambdaEvent<Value>) -> Result<Value, Error> {
    let (event, _context) = event.into_parts();
    let first_name = event["firstName"].as_str().unwrap_or("world");

    Ok(json!({ "message": format!("Hello, {}!", first_name) }))
}
```
This code has 2 methods. The `main()` which initializes some lambda related scaffolding. And the `func(event: LambdaEvent<Value>)` which is our code's handler function. This is where all the business logic for our lambda lives.

You'll notice that the [AWS Lambda Rust Runtime](https://github.com/awslabs/aws-lambda-rust-runtime/tree/v0.5.0) uses [tokio](https://tokio.rs/) so the `main` function is preceded with a tokio macro and our handler function `func` needs to be `async`. The details of tokio are out of scope for this article. For now it is enough to just know it is an async framework used in some rust projects.


## Fix the copy pasta errors

Currently the lambda will not build, your IDE may show some errors related to crates not found.

Lets update the `Cargo.toml` with the required libraries.

I use [Cargo Edit](https://github.com/killercup/cargo-edit) which extends Cargo with commands to manage dependencies without having to manually edit your `Cargo.toml`

```
cargo add lambda_runtime

cargo add serde_json

cargo add tokio
```
Here we have added the [AWS Lambda Rust Runtime](https://github.com/awslabs/aws-lambda-rust-runtime/tree/v0.5.0), [Serde JSON](https://github.com/serde-rs/json), and [tokio](https://tokio.rs/)

Finally, the project should be ready to build.

## How to build
We will need to build a binary for the lambda environment, which might be different from our laptop. I'm using MacOS, so I'll need to cross compile. If you are using linux you might be able to build for our target architecture without these extra steps. You are are using a mac, you can follow along to add the zig linker and appropriate target for the Lambda environment.

### Check that normal local builds are working, and fix any dependency issues if needed.

Before we get started with the cross compile steps, lets make sure a normal build works without and issues.

```
cargo build
```
Ok that should succeed and let us know that the code and dependencies are correct.

### Build for AWS Lambda install x86 target

We are going to target the "Amazon Linux2 x86" for our Lambda environment. In order to achieve this from a MacOS we will need to cross compile.  The [AWS Lambda Rust Runtime](https://github.com/awslabs/aws-lambda-rust-runtime/tree/v0.5.0#1-cross-compiling-your-lambda-functions) Github repo has instructions for cross compiling that we will follow here. They use `zig` and a cargo binary `zigbuild` to accomplish the build process.

1. First install [Zig](https://ziglang.org/)

```
brew install zig
```

2. Install cargo [zigbuild](https://github.com/messense/cargo-zigbuild)
```
cargo install cargo-zigbuild
```

3. Add the target for x86
```
rustup target add x86_64-unknown-linux-gnu
```

4. Finally, we build the release

```
cargo zigbuild --release --target x86_64-unknown-linux-gnu
```

## Package and Create the Lambda

Ok, now we have a binary that we can run on the AWS Lambda infrastructure. Next step, we need to bundle the binary and upload to our AWS account.

### Create a zip archive

In order to install and test our lambda, we are going to create a zip file with the binary. The binary is expected to have the name `"bootstrap"`, so the following command renames it and creates a zip file that is ready to be uploaded to AWS.

```
cp ./target/x86_64-unknown-linux-gnu/release/lambda_test ./bootstrap && zip lambda.zip bootstrap && rm bootstrap
```

### Create new function
`Note: Currently, I'm missing details about how to create the role and copy the ARN`

Now that we have the zip file with the "bootstrap" binary, we are ready to create a new lambda function.  In the following command we use the `aws` cli to create a new lambda function with the name "rustTest". 

```
aws lambda create-function --function-name rustTest \
  --handler doesnt.matter \
  --zip-file fileb://./lambda.zip \
  --runtime provided.al2 \
  --role arn:aws:iam::{PUT_AWS_ACCT}:role/lambda_basic_execution \
  --environment Variables={RUST_BACKTRACE=1} \
  --tracing-config Mode=Active
  ```
`Note` you will need to modify this command with your own `"--role"` name including account number.

Once we have run this command the new rust lambda function exists in our AWS account and we are ready to run a test.

## Test and Check the logs

### Test invoke
We can invoke for testing directly from the cli using this cli command.
```
aws lambda invoke \
  --cli-binary-format raw-in-base64-out \
  --function-name rustTest \
  --payload '{"firstName": "James"}' \
  output.json
```

### Check output
The output of the lambda is written to a file specified in the "invoke" command.
```
> cat output.json
{"message":"Hello, James!"}%
```

### Check logs
`Note: would be good to add how to check cloudwatch logs from cli`

After invoking our "rustTest" lambda, we will want to check the cloudwatch logs and see the execution time.

```
START RequestId: 2b0a5ad0
Version: $LATEST
END RequestId: 2b0a5ad0
REPORT RequestId: 2b0a5ad0	Duration: 0.85 ms	Billed Duration: 1 ms	Memory Size: 128 MB	Max Memory Used: 15 MB	
XRAY TraceId: 1-6213be54	SegmentId: 282cd	Sampled: true	
```
The "hello world" lambda doesn't do much, but it is still impressive to see a `Billed Duration: 1 ms` 

## Conclusions

We were able to successfully create a Lambda using Rust! Using the [AWS Lambda Rust Runtime](https://github.com/awslabs/aws-lambda-rust-runtime) made this easy, the code is straight forward. And using Zig made it easy to cross compile. The final result is a small binary that runs super fast.

#### Finally the Source Code
 The complete project is here in my github repo [https://github.com/millerjam/rust_lambda_hello_world](https://github.com/millerjam/rust_lambda_hello_world)


## Next Time
Next time, we will look at moving past "hello world" by adding some more functionality that we would need for any real world projects. 


---
title: "Rust Lambda Hello World Logging"
date: 2022-03-13T14:35:38-04:00
draft: false
---

In my previous article, I started a tutorial on creating a lambda in Rust. If you missed it, you can catch up on that article here -> [Rust Lambda Hello World](../rust-lambda-hello-world) 

## Lets add some logging 

Before we go much further with our Rust code, let's add some logging. We are take advantage of the [tracing library](https://docs.rs/tracing/latest/tracing/) which is already integrated into many of the lambda dependencies.


>    Details about the tracing library are out of scope for this article, but you can learn more from the [tracing crate docs](https://docs.rs/tracing/latest/tracing/)


### Init the logger

Setting up the logger requires initializing the tracing subscriber library, the [AWS Lambda Rust Runtime Examples](https://github.com/awslabs/aws-lambda-rust-runtime/tree/v0.5.0/lambda-runtime/examples) has a examples with some details about how to get started.

Here is the code that we will have to add:
```rust
fn init_lambda_tracing() {
    tracing_subscriber::fmt()        
        .with_max_level(tracing::Level::INFO)
        // this needs to be set to false, otherwise ANSI color codes will
        // show up in a confusing manner in CloudWatch logs.
        .with_ansi(false)
        // disabling time is handy because CloudWatch will add the ingestion time.
        .without_time()
        .init();
}
```

I have created a small function `init_lambda_tracing()` with all the details for setting up the logging. This function will setup the default `tracing_subscriber` which prints logs to stdout. 
The additional methods called on the builder, set the default logging level to "info" and disable some of the defaults to make things nicer in the Lambda environment. Finally, I added
additional code to the `main()` to call this function before the lambda runtime setup.

### Fix the compile issues

Even though we did not add any additional `import ...` lines, we still need to add our new tracing library dependencies to `Cargo.toml`, before our code will build.

>The reason we did not add `import..` statements is that we fully qualified the crate directly in the code with the prefix `tracing_subscriber::...`.

Lets go ahead and add the 2 dependencies: [tracing](https://crates.io/crates/tracing), and [tracing-subscriber](https://crates.io/crates/tracing-subscriber)

```
cargo add tracing

cargo add tracing-subscriber

```

## Lets test it!

Ok now that we have logging setup, let's add some logs and try it. The tracing library provides logging macros such as, `debug!`, `info!`, `warn!`, `error!`

We will add an new "info" log  to the `func` method.

```rust
info!("Going to say hello to {}!", first_name);
```

We will also need to import
```rust
use tracing::info;
```

Now we can update the function and test.

### Compile & update the function
Just like in the previous article, we need to compile for the lambda env and update the function.

#### Compile
```
cargo zigbuild --release --target x86_64-unknown-linux-gnu
```

#### Update the existing function
```
aws lambda update-function-code --function-name rustTest \
  --zip-file fileb://./lambda.zip
  ```

## Test invoke & check the logs

Now that the lambda is update with the new code, we can go ahead and test and check for our new logs.
### Invoke 
```
aws lambda invoke \
  --cli-binary-format raw-in-base64-out \
  --function-name rustTest \
  --payload '{"firstName": "James"}' \
  output.json
```

### Now check the cloudwatch logs again

```log
START RequestId: 93bcbe8c-4b2e-4af3-b00d-3b4b27083ebe Version: $LATEST
INFO lambda_test: going to say hello to James
END RequestId: 93bcbe8c-4b2e-4af3-b00d-3b4b27083ebe
REPORT RequestId: 93bcbe8c-4b2e-4af3-b00d-3b4b27083ebe	Duration: 1.16 ms	Billed Duration: 33 ms	Memory Size: 128 MB	Max Memory Used: 17 MB	Init Duration: 31.20 ms	
XRAY TraceId: 1-622553a3-5644934610a4952626f10403	SegmentId: 3b7b79993cba7894	Sampled: true	
```

Success! We now have our log, 

```log
INFO lambda_test: going to say hello to James
```


## Reconfigure the logger

### Update the code for dynamic log levels
Now that we have logging working, let's reconfigure it so that it reads the log level
from an environment variable. This is will be much more flexible and give us the ability to turn logging levels up and down while we are developing and debugging, without having to recompile the code.

We can update the `init_lambda_tracing` like this:
```rust
fn init_lambda_tracing() {
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::from_default_env())
        // this needs to be set to false, otherwise ANSI color codes will
        // show up in a confusing manner in CloudWatch logs.
        .with_ansi(false)
        // disabling time is handy because CloudWatch will add the ingestion time.
        .without_time()
        .init();
}
```

We replaced the hardcoded `set_max_level...` with 
```rust
.with_env_filter(EnvFilter::from_default_env())
``` 
this will read the logging level from the 
environment variable `RUST_LOG`. Now we can change the log level dynamically without having the recompile and update the running code.

We will also have to update the `Cargo.toml` to make sure the "EnvFilter" feature is available from the `tracing_subscriber` library. 

```Cargo.toml
tracing-subscriber = {version = "0.3.9", features = ["env-filter"]}
```

### Compile & update the function configuration

Now that code is update, we need to rebuild, update the function, and then also the config
to add the new `"RUST_LOG"` environment variable. Let's set the logging level to "DEBUG" for testing.

#### Compile
```
cargo zigbuild --release --target x86_64-unknown-linux-gnu
```

#### Update the existing function
```
aws lambda update-function-code --function-name rustTest \
  --zip-file fileb://./lambda.zip
  ```

#### Update the existing function configuration
```
aws lambda update-function-configuration \
    --function-name  rustTest \
    --environment Variables="{RUST_BACKTRACE=1,RUST_LOG=debug}"
```
### Test invoke again

```
aws lambda invoke \
  --cli-binary-format raw-in-base64-out \
  --function-name rustTest \
  --payload '{"firstName": "James"}' \
  output.json
```
### Check the cloudwatch logs again
Logs are much more verbose, we can see lots of new logs from the included dependencies.

```
START RequestId: 9ebad5dc-3cee-4302-a0f8-0e592e432f41 Version: $LATEST
DEBUG hyper::client::connect::http: connecting to 127.0.0.1:9001
DEBUG hyper::client::connect::http: connected to 127.0.0.1:9001
DEBUG hyper::proto::h1::io: flushed 109 bytes
DEBUG hyper::proto::h1::io: parsed 7 headers
DEBUG hyper::proto::h1::conn: incoming body is content-length (21 bytes)
DEBUG hyper::proto::h1::conn: incoming body completed
DEBUG hyper::client::pool: pooling idle connection for ("http", 127.0.0.1:9001)
INFO lambda_test: going to say hello to James
DEBUG hyper::client::pool: reuse idle connection for ("http", 127.0.0.1:9001)
DEBUG hyper::proto::h1::io: flushed 198 bytes
DEBUG hyper::proto::h1::io: parsed 3 headers
DEBUG hyper::proto::h1::conn: incoming body is content-length (16 bytes)
DEBUG hyper::client::connect::http: connecting to 127.0.0.1:9001
DEBUG hyper::proto::h1::conn: incoming body completed
DEBUG hyper::client::pool: reuse idle connection for ("http", 127.0.0.1:9001)
DEBUG hyper::proto::h1::io: flushed 109 bytes
END RequestId: 9ebad5dc-3cee-4302-a0f8-0e592e432f41
REPORT RequestId: 9ebad5dc-3cee-4302-a0f8-0e592e432f41	Duration: 1.46 ms	Billed Duration: 38 ms	Memory Size: 128 MB	Max Memory Used: 17 MB	Init Duration: 36.11 ms	
XRAY TraceId: 1-622551be-6d58480b496655162937c59c	SegmentId: 03a6c5576dc28250	Sampled: true	
DEBUG hyper::client::connect::http: connected to 127.0.0.1:9001
DEBUG hyper::client::pool: pooling idle connection for ("http", 127.0.0.1:9001)
```

## Conclusions

We were able to successfully add logging to our "hello world" lambda, and make it possible to dynamically change the logging level based on an environment variable. Using the tracing library made this easy, and the code is straight forward.

#### Finally the Source Code
 The complete project is here in my github repo [https://github.com/millerjam/rust_lambda_hello_world](https://github.com/millerjam/rust_lambda_hello_world)


## Next Time
Next time, we will look at adding a unit test and finally try switching from building x86 to ARM for even more savings on our lambda executions.

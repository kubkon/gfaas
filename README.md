<div align="center">
  <h1><code>gfaas</code></h1>

  <p>
    <strong>Function-as-a-Service on top of Golem Network using gWasm platform.</strong>
  </p>

  <p>
    <a href="https://github.com/golemfactory/gfaas/actions?query=branch%3Amaster"><img src="https://github.com/golemfactory/gfaas/workflows/Continuous%20Integration/badge.svg" alt="Build status" /></a>
    <a href="https://docs.rs/gfaas"><img src="https://docs.rs/gfaas/badge.svg" alt="Documentation Status" /></a>
  </p>

  <h3>
    <a href="https://golemfactory.github.io/gfaas/gfaas/">Docs (master)</a>
  </h3>
</div>

This crate allows you to distributed heavy-workload functions to the [Golem Network] (or any
other compatible backend when support for it is added in the future).

[Golem Network]: https://golem.network/

## Quick start

The usage is pretty straightforward. In your `Cargo.toml`, put `gfaas` as your dependency

```toml
# Cargo.toml
[dependencies]
gfaas = "0.3"
```

You can now annotate some heavy-workload function to be distributed on the Golem Network
like so

```rust,ignore
use gfaas::remote_fn;

#[remote_fn]
fn hello(input: String) -> String {
    // let's make the input all-caps
    input.to_uppercase().to_string()
}

#[actix_rt::main]
async fn main() {
    let input = "hey there gfaas";
    let output = hello("hey there gfaas".to_string()).await.unwrap();
    assert_eq!(input.to_uppercase(), output)
}
```

In order to compile your code you'll need to use our custom wrapper on top of `cargo` called
`gfaas`. You can install the tool with `cargo-install` like so

```
cargo install gfaas-cli
```

Then, you can use `gfaas` like you would use `cargo`. So, to build and run on Golem Network,
you'd execute

```
gfaas run
```

## Notes about `gfaas::remote_fn`

When you annotate a function with `gfaas::remote_fn` attribute, it gets expanded into a
full-fledged async function which is fallible. So for instance, the following function

```rust,ignore
use gfaas::remote_fn;

#[remote_fn]
fn hello(input: String) -> String;
```

expands into

```rust,ignore
async fn hello(input: String) -> Result<String, gfaas::Error>;
```

Therefore, it is important to remember that you need to run the function in an async block
and in order to get the result of your function back, you need to unpack it from the outer
`Result` type.

The asyncness and the `Result` are there due to the nature of any distributed computation
run on top of some network of nodes: it may fail due to reasons not related to your app
such as network downtime, etc.

Furthermore, the input and output arguments of your function have to be serializable, and
so they are expected to derive `serde::Serialize` and `serde::Deserialize` traits.

### Specifying Golem's configuration parameters

You can currently set the following configuration parameters directly via `gfaas::remote_fn`
attribute:

* (maximum) budget in NGNT (defaults to 100):

```rust,ignore
#[remote_fn(budget = 100)]
fn hello(input: String) -> String;
```

* timeout in seconds (defaults to 10 minutes):

```rust,ignore
#[remote_fn(timeout = 600)]
fn hello(input: String) -> String;
```

* subnet tag (defaults to "devnet-alpha.2"):

```rust,ignore
#[remote_fn(subnet = "devnet-alpha.2")]
fn hello(input: String) -> String;
```

Of course, nobody stops you from setting any number of parameters at once

```rust,ignore
#[remote_fn(budget = 10, subnet = "my_subnet")]
fn hello(input: String) -> String;
```

## Notes about `gfaas` build tool and adding dependecies for your functions

The reason that a custom wrapper around `cargo` is needed, is because the function
annotated with `gfaas::remote_fn`, under-the-hood is actually automatically cross-compiled
into a WASI binary.

In addition, since the functions are cross-compiled to WASI, you need to install
`wasm32-wasi` target in your used Rust toolchain. Furthermore, for that same reason, not
all crates are compatible with WASI yet, but you can manually specify which crates you
want your functions to depend on by adding a `[gfaas_dependencies]` section to your `Cargo.toml`

```toml
# Cargo.toml
[package]
author = "Jakub Konka"

[dependecies]
actix = "1"

[gfaas_dependencies]
log = "0.4"
```

## Notes on running your app locally (for testing)

It is well known that prior to launching our app on some distributed network of nodes, it
is convenient to first test the app locally in search of bugs and errors. This is also
possible with `gfaas`. In order to force your app to run locally, simply pass
`run_local=true` as argument to `gfaas::remote_fn` attribute

```rust,ignore
#[remote_fn(run_local = true)]
fn hello(input: String) -> String;
```

This will spawn all of your annotated functions in separate threads on your machine locally,
so you can verify that everything works as expected prior to launching the tasks on the
Golem Network.

## Examples

A couple illustrative examples of how to use this crate can be found in the `examples/`
directory. All examples require `gfaas` build tool to be built.

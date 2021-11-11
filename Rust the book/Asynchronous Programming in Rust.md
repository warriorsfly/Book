# Asynchronous Programming in Rust

## 1. Getting Started

Welcome to Asynchronous Programming in Rust! If you're looking to start writing asynchronous Rust code, you've come to the right place. Whether you're building a web server, a database, or an oprating system, this book will show you how to use Rust's asynchronous programming tools to get the most out of your hardware.

**What This Book Covers**

This book aims to be a comprehensive, up-to-date guide to using Rust's async language features and libraries, appropriate for beginners and old hands alike.

- The early chapters provide an introduction to async programming in general ,and to Rust's particular take on it.
- The middle chapters discuss key utilities and control-flow tools you can use when writing async code, and describe best-practices for structuring libraries and applications to maximize performance and reusablity.
- The last section of the book covers the broader async ecosystem, and provides a number of examples of how to accomplish common tasks.

With that out of the day, let's explore the exciting world of Asynchronous Programming in Rust!

### 1.1 Why Async?

We all love how Rust empowers us to write fast, safe software. But how does asynchronous programming fit into this vision?

Asynchronous programming, or async for short, is a *concurrent programming model* supported supported by an increasing number of programming languages. It lets you run a large number of concurrent tasks on a small number of OS threads, while preserving much of the look and feel of ordinary synchronous programming, through `async/await` syntax.

**Async vs other concurrency models**

Concurrent programming is less mature and "standardized" than regular, sequential programming. As a result, we express concurrency differently depending on which concurrent programming model the language is supporting. A brief overview of the most popular concurrency models can help you understand how asynchronous programming fits within the broader field of concurrent programming:

- **OS threads** don't require any changes to the programming model, which makes it very easy to express concurrency. However, synchronizing between threads can be difficult, and the performance overhead is large. Thread pools can mitigrate some of these costs, but not enough to support massive IO-bound workloads.
- **Event-driven programming**, in conjunction with *callbacks*, can be very performant, but tends to result in a verbose, "non-linear" control flow. Data flow and error propagation is often hard to follow.
- **Coroutines**, like threads, don't require changes to the programming model, which makes them easy to use. Like async, they can also support a large number of tasks. However, they abstract away low-level details that are important for systems programming and custom runtime implementors.
- **The actor model** divides all concurrent computation into units called actors, which communicate through fallible message passing, much like in distributed systems. The actor model can be efficiently implemented, but it leaves many practical issues unanswered, such as flow control and retry loginc.

In summary, asynchronous programming allows highly performant implementations that are suitable for low-level languages like Rust, while providing most of the ergonomic benefits of threads and coroutines.

**Async in Rust vs other language**

The primary alternative to async in Rust is using OS threads, either directly through `std::thread` or indirectly through a thread pool. Migrating from threads to async or vice versa typically requires major refactoring work, both in terms of implementation and (if you are building a library) any exposed public interfaces. As such, picking the model that suits your needs early can save a lot of development time.

**OS threads** are suitable for a small number of tasks, since threads come with CPU and memory overhead. Spawning and swithcing between threads is quite expensive as even idle threads consume system resources. A thread pool library can help mitigate some of these costs, but not all. However, threads let you reuse existing synchronous code without significant code changes-no particular programming model is required. In some operationg systems ,you can also change the prority of a thread, which is useful for drivers and other latency sensitive applications.

**Async** provides significantly reduced CPU and memory overhead, especially for workloads with a large amount of IO-bound tasks, such as servers and databases. All else equal, you can have orders of magnitude more tasks thanOA threads, because an async runtime uses a small amount of (expensive) threads to handle a large amount of (cheap) tasks. However ,async Rust results in larger binary blobs due to the state machines generated form async functions and since each executable bundles an async runtime.

On a last note, asynchronous programming is not *better* than threads, but different. If you don't need async for performance reasons, threads can often be the simpler alternative.

**Example: Concurrent downloading**

In this example our goal is to download two web pages concurrently. In a typical threaded application we need to spawn threads to achieve concurrency.

```rust
fn get_two_sites(){
  // Spawn two threads to do work.
  let t1 = thread::spawn(||download("https://www.foo.com"));
  let t2 = thread::spawn(||download("https:??www.bar.com"));
  //Wait for both threads to complete.
  t1.join().expect("t1 panicked");
  t2.join().expect("t1 panicked");
}
```

However, downloading a web page is a small task; creating a thread for such a small amount of work is quite wasteful. For a large application, it can easily become a bottleneck. In async Rust, we can run these tasks concurrently without extra threads:

```rust
async fn get_two_sites_async() {
    // Create two different "futures" which, when run to completion,
    // will asynchronously download the webpages.
    let future_one = download_async("https://www.foo.com");
    let future_two = download_async("https://www.bar.com");

    // Run both futures to completion at the same time.
    join!(future_one, future_two);
}
```

Here, no extra threads are created. Additionally, all function calls are statically dispatched，and there are no heap allocations! However, we need to write the code to be asynchronous in the first place, which this book will help you achieve.

**Custom concurrency models in Rust**

On a last note, Rust doesn't force you to chosse between threads and async. You can use both models within the same application, which can be useful when you have mixed threaded and async dependencies. In fact, you can even use a different concurrency model altogether, such as event-driven programming, as long as you  find a library that implements it.

### 1.2 The State of Asynchronous Rust

Parts of async Rust are supported with the same stability guarantees as synchronous Rust. Other parts are still maturing and will change over time. With async Rust, you can expect:

- Outstanding runtime performence for typical concurrent workloads.
- More frequent interaction with advanced language features, such as lifetimes and pinning.
- Some compatibility constraints, both between sync and async code, and between different async runtimes.
- Higher maintenance burden, due to the ongoing evolution of async runtimes and language support.

In short, async Rust is more difficult to use and can result in a higher maintenance burden than synchronous Rust, but gives you best-in-class performance in return. All areas of async Rust are constantly improving, so the impact or these issues will wear off over time.

**Language and library support**

While asynchronous programming is supported by Rust itself, most async applications depend on functionality provided by community crates. As such, you need to rely on a mixture of language features and library support:

- The most fundamental traits, types and functions, such as the `Future` trait are provided by the standard library.
- The `async/await` syntax is supported directly by the Rust compiler.
- Many utility types, macros and functions are provided by the `futures` crate. They can be used in any async Rust application.
- Execution of async code, IO and task spawning are provided by "async runtimes", such as Tokio and async-std. Most async applications, and some async crates, depend on a specific runtime. See " [The Async Ecosystem](https://rust-lang.github.io/async-book/08_ecosystem/00_chapter.html)" section for more details.

Some language features you may be used to from asynchronous Rust are not yet available in async Rust. Notably, Rust does not let you declare async functions in traits. Instead, you need to use workarounds to achieve the same result, which can be more verbose.

**Compiling and debugging**

For the most part, compiler-and runtime errors in async Rust work the same way as they have always done in Rust. There are a few noteworthy differences:

> **Compilation errors**
>
> Compilation errors in async Rust conform to the same high standards as synchronous Rust, but since async Rust often depends on more complex language features, such as lifetimes and pinning, you may encounter these types of errors more frequently.
>
> **Runtime error**
>
> Whenever the compiler encounters an async function, it generates a state machine under the hood. Stack traces in async Rust typically contain details from these state machines, as well as function calls form the runtime. As such, interpreting stack traces can be a bit more involved than it would be in synchronous Rust.
>
> **New failure modes**
>
> A few novel failure modes are possible in async Rust, for instance if you call a blocking function from an async context or if you  implement the `Future` trait incorrectly. Such errors can silently pass both the compiler and sometimes even unit tests. Having a firm understanding of the underlying concepts, which this book aims to give you , can help you avoid these pitfalls.

**Compatibility considerations**

Asynchronous and synchronous code cannot always be combined freely. For instance, you can't directly call an async function from a sync function, Sync and async code also tend to promote fifferent design patterns, which can make it difficult to compose code intended for the different environments.

Even async code cannot always be combined freely. Some crates depend on a specific async runtime to function. If so, it is usually specified in the crate's dependency list.

These compatibility issues can limit your options, so make sure to research which async runtime and what crates you may need early. Once you have settled in with a runtime you won't have to worry much about compatibility.

**Performance characteristics**

The performance of async Rust depends on the implementation of the async runtime you're using. Even though the runtimes that power async Rust applications are relatively new , they perform exceptionally well for most practical workloads.

That said ,most of the async ecosystem assumes a *multi-threaded* runtime. This makes it difficult to enjoy the theoretical performance benefits of single-threaded async applications, namely cheaper synchronization. Another overlooked use-case is *latency sensitive tasks*, which are important for drivers, GUI applications and so on. Such tasks depend on runtime and/or OS support in order to be scheduled appropriately. You can expect better library support for these use cases in the future.

### 1.3 async/.await **Primer**

`async`/`.await` is Rust's built-in tool for writing asynchronous functions that look like asynchronous code. `async` transforms a block of code into state machine that implements a trait called `Future`. Whereas calling a blocking function in a synchronous method would block the whole thread, blocked `Future`s will yield control of the thread, allowing other `Future`s to run.

Let's add some dependencies to the `Cargo.toml` file:

```toml
[dependencies]
futures = "0.3"
```

To create an asynchronous function, you can use the `async fn` syntax:

```rust
async fn do_something(){/*...*/}
```

The value returned by `async fn` is a `Future`. For anything to happen, the `Future` needs to be run on an executor.

```rust
// `block_on` blocks the current thread until the provided future has run to
// completion. Other executors provide more complex behavior, like scheduling
// multiple futures onto the same thread.
use futures::executor::block_on;

async fn hello_world() {
    println!("hello, world!");
}

fn main() {
    let future = hello_world(); // Nothing is printed
    block_on(future); // `future` is run and "hello, world!" is printed
}
```

Inside an `async fn`, you can use `.await` to wait for the completion of another type that implements the `Future` trait, such as the output of another `async fn`. Unlike `block_on`,`.await` doesn't;t block the current thread, but instead asynchronously waits for the future to complete, allowing other tasks to run if the future is currently unable to make progress.

For example, imagine that we have three `async fn`: `learn_song`, `sing_song` and `dance`:

```rust
async fn learn_song()->Song{/*...*/}
async fn sing_song(song:Song){/*...*/}
async fn dance(){/*...*/}
```

One way to do learn, sing and dance would be to block on each of these individually:

```rust
fn main(){
  let song = block_on(learn_song());
  block_on(sing_song(song));
  block_on(dance());
}
```

However, we're not giving the best performance possible this way-we're only ever doing one thing at once! Clearly we have to learn the song before we can sing it, but it's possible to dance at the same time as learning and singing the song. To do this, we can create two separate `async fn` which can be run concurrently:

```rust
async fn learn_and_sing(){
   // Wait until the song has been learned before singing it.
    // We use `.await` here rather than `block_on` to prevent blocking the
    // thread, which makes it possible to `dance` at the same time.
    let song = learn_song().await;
    sing_song(song).await;
}

async fn async_main(){
  let f1=learn_and_sing();
  let f2=dance();
  // `join!` is like `.await` but can wait for multiple futures concurrently.
  // If we're temporarily blocked in the `learn_and_sing` future, the `dance`
  // future will take over the current thread. If `dance` becomes blocked,
  // `learn_and_sing` can take back over. If both futures are blocked, then
  // `async_main` is blocked and will yield to the executor.
  futures::join!(f1,f2);
}

fn main(){
  block_on(async_main());
}
```

In this example ,learning the song must happen before singing the song, but both learning and singing can happen at the same time as dancing. If we used `block_on(learn_song())` rather than `lean_song().await` in `learn_and_sing()`, the thread wouldn't be able to do anything else while `learn_song` was running. This would make it impossible to dance at the same time. By `.await`-ing the `learn_song` future, we allow other tasks to take over the current thread if `learn_song` is blocked. This makes it possible to run multiple futures to completion concurrently on the same thread.

## 2. Under the Hood: Executing Futures and Tasks

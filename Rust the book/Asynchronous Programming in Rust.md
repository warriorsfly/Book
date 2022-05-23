# Asynchronous Programming in Rust

## 1. Getting Started

Welcome to Asynchronous Programming in Rust! If you're looking to start writing asynchronous Rust code, you've come to the right place. Whether you're building a web server, a database, or an operating system, this book will show you how to use Rust's asynchronous programming tools to get the most out of your hardware.

**What This Book Covers**

This book aims to be a comprehensive, up-to-date guide to using Rust's async language features and libraries, appropriate for beginners and old hands alike.

- The early chapters provide an introduction to async programming in general ,and to Rust's particular take on it.
- The middle chapters discuss key utilities and control-flow tools you can use when writing async code, and describe best-practices for structuring libraries and applications to maximize performance and reusability.
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
- **The actor model** divides all concurrent computation into units called actors, which communicate through fallible message passing, much like in distributed systems. The actor model can be efficiently implemented, but it leaves many practical issues unanswered, such as flow control and retry logic.

In summary, asynchronous programming allows highly performant implementations that are suitable for low-level languages like Rust, while providing most of the ergonomic benefits of threads and coroutines.

**Async in Rust vs other language**

The primary alternative to async in Rust is using OS threads, either directly through `std::thread` or indirectly through a thread pool. Migrating from threads to async or vice versa typically requires major refactoring work, both in terms of implementation and (if you are building a library) any exposed public interfaces. As such, picking the model that suits your needs early can save a lot of development time.

**OS threads** are suitable for a small number of tasks, since threads come with CPU and memory overhead. Spawning and switching between threads is quite expensive as even idle threads consume system resources. A thread pool library can help mitigate some of these costs, but not all. However, threads let you reuse existing synchronous code without significant code changes-no particular programming model is required. In some operating systems ,you can also change the priority of a thread, which is useful for drivers and other latency sensitive applications.

**Async** provides significantly reduced CPU and memory overhead, especially for workloads with a large amount of IO-bound tasks, such as servers and databases. All else equal, you can have orders of magnitude more tasks than OS threads, because an async runtime uses a small amount of (expensive) threads to handle a large amount of (cheap) tasks. However ,async Rust results in larger binary blobs due to the state machines generated form async functions and since each executable bundles an async runtime.

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

On a last note, Rust doesn't force you to choose between threads and async. You can use both models within the same application, which can be useful when you have mixed threaded and async dependencies. In fact, you can even use a different concurrency model altogether, such as event-driven programming, as long as you  find a library that implements it.

### 1.2 The State of Asynchronous Rust

Parts of async Rust are supported with the same stability guarantees as synchronous Rust. Other parts are still maturing and will change over time. With async Rust, you can expect:

- Outstanding runtime performance for typical concurrent workloads.
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

Asynchronous and synchronous code cannot always be combined freely. For instance, you can't directly call an async function from a sync function, Sync and async code also tend to promote different design patterns, which can make it difficult to compose code intended for the different environments.

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

In this section, we'll cover the underlying structure of how `Future`s and asynchronous tasks are scheduled. If you're only interested in learning how to write higher-level code that uses existing `Future` types and aren't interested in the details of how `Future` types work, you can skip ahead to the `async`/`await` chapter. However, several of the topics discussed in this chapter are useful for understanding how `async`/`await` code works, understanding the runtime and performance properties of `async`/`await` code, and building new asynchronous primitives. If you decide to skip this section now, you may want to bookmark it to revisit in the future.

Now, with that out of the way, let's talk about the `Future` trait.

### 2.1 The Future Trait

The `Future` trait is at the center of asynchronous programming in Rust. A `Future` is an asynchronous computation that can produce a value(although that value may be empty, e.g.`()`). A *simplified* version of the future trait might look something like this:

```rust
trait SimpleFuture{
  type Output;
  fn poll(&mut self, wake:fn())->Poll<Self::Output>;
}

enum Poll<T>{
  Ready(T),
  Pending,
}
```

Futures can be advanced by calling the `poll` function, which will drive the future as far towards completion as possible. If the future completes, it returns `Poll::Ready(result)`. If the future is not able to complete yet, it returns `Poll::Pending` and arranges for the `wake()` function to be called when the `Future` is ready to make more progress. When `wake()` is called, the executor driving the `Future` will call `poll` again so that the `Future` can make more progress.

Without `wake()`, the executor would have no way of knowing when a particular future could make progress, and would have to be constantly polling every future. With `wake()`, the executor knows exactly which futures are ready to be `poll` ed.

For example, consider the case where we want to read from a socket that may or may not have data available already. If there is data, we can read it in and return `Poll:Ready(data)`, but if no data is ready, our future is blocked and can no longer make progress. When no data is available, we must register `wake` to be called when data becomes ready on the socket, which will tell the executor that our future is ready to make progress. A simple `SocketRead` future might look something like this:

```rust
pub struct SocketRead<'a>{
  socket: &'a Socket,
}

impl SimpleFuture for SocketRead<'_>{
  type Output=Vec<u8>;
  fn poll(&mut self,wake:fn())->Poll<Self::Output>{
    if self.socket.has_data_to_read(){
      Poll::Ready(self.socket.read_buf())
    }else{
      self.socket.set_readable_callback(wake);
      Poll::Pending
    }
  }
}
```

The model of `Future`s allows for composing together multiple asynchronous operations without needing intermediate allocations. Running multiple futures at once or chaining futures together can be implemented via allocation-free state machines, like this:

```rust
/// A SimpleFuture that runs two futures to completion, one after another.
//
// Note: for the purposes of this simple example, `AndThenFut` assumes both
// the first and second futures are available at creation-time. The real
// `AndThen` combinator allows creating the second future based on the output
// of the first future, like `get_breakfast.and_then(|food| eat(food))`.
pub struct AndThenFut<FutureA,FutureB>{
  first:Option<FutureA>,
  second:FutureB,
}

impl<FutureA,FutureB> SimpleFuture for AndThenFut<FutureA,FutureB>
where
	FutureA:SimpleFuture<Output=()>,
	FutureB:SimpleFuture<Output=()>,
{
  type Output=();
  fn poll<&mut self,wake:fn())->Poll<Self::Output>{
    if let Some(first)=&mut self.first{
      match first.poll(wake){
        Poll::Ready(())=>self.first.take(),
        Poll::Pending=> return Poll::Pending,
      };
    }
    self.second.poll(wake)
	}
}
```

These examples show how the `Future` trait can be used to express asynchronous control flow without requiring multiple allocated objects and deeply nested callbacks. With the basic control-flow out of the way, let's talk about the real `Future` trait and how it is different.

```rust
trait Future{
  type Output;
  fn poll(
  //Note the change from `&mut self` to `Pin<&mut Self>`
    self: Pin<&mut Self>,
    cx:&mut Context<'_>,
  )->Poll<Self::Output>;
}
```

The first change you'll notice is that our `self` type is no longer `&mut Self`, but has changed to `Pin<&mut Self>`. We'll talk more about pinning in a later section, but for now know that it allows us to create futures that are immovable. Immovable objects can store pointers between their fields, e.g. `struct MyFut { a:i32, ptr_to_a: *const i32}`. Pinning is necessary to enable async/await.

Secondly, `wake:fn()` has changed to `&mut Context<'_>`. In `SimpleFuture`, we used a call to a function pointer(`fn()`) is just a function pointer, it can't store any data about *which* `Future` called `wake`.

In a real-world scenario, a complex application like a web server may have thousands of different connections whose wakes should all be managed separately. The `Context` type sovles this by providing access to a value of type `Waker`, which can be used to wake up a specific task.

### 2.2 Task Wakes with Waker

It's common that futures aren't able to complete the first time they are `poll`ed. When this happens, the future needs to ensure that it is polled again once it is ready to make more progress. This is done with the `Waker` type.

Each time a future is polled, it is polled as part of a "task". Tasks are the top-level futures that have been submitted to an executor.

`Waker` provides a `wake()` method that can be used to tell the executor that the associated task should be awoken. When `wake()` is called, the executor knows that the task associated with the `Waker` is ready to make progress, and its future should be polled again.

`Waker` also implements `clone()` so that it can be copied around and stored.

Let's try implementing a simple timer future using `Waker`.

**Applied: Build a Timer**

For the sake of the example, we'll just spin up a new thread when the timer is created, sleep for the required time, and then signal the timer future when the time, and then signal the timer future when the time window has elapsed.

First, start a new project with `cargo new --lib timer_future` and add the imports we'll need to get started to `src/lib.rs`:

```rust
use std::{
  future::Future,
  pin::Pin,
  sync::{Arc,Mutex},
  task::{Context,Poll,Waker},
  thread,
  time::Duration,
};
```

Let's start by defining the future type itself. Our future needs a way for the thread to communicate that the timer has elapsed and the future should complete. We'll use a shared `Arc<Mutex<..>>` value to communicate between the thread and the future.

```rust
pub struct TimerFuture{
  shared_state:Arc<Mutex<SharedState>>,
}

/// Shared state between the future and the waiting thread
struct ShareState{
  /// Whether or not the sleep time has elapsed
  completed:bool,
/// The waker for the task that `TimerFuture` is running on.
    /// The thread can use this after setting `completed = true` to tell
    /// `TimerFuture`'s task to wake up, see that `completed = true`, and
    /// move forward.
    waker: Option<Waker>,
}
```

Now, let's actually write the `Future` implemention~

```rust
impl Future for TimerFuture{
  type Output=();
  fn poll(self:Pin<&mut Self>, ctx:&mut Context<'_>)->Poll<Self::Output>{
    let mut shared_state = self.shared_state.lock().unwrap();
    if shared_state.complated{
      // Look at the shared state to see if the timer has already completed.
      return Poll::Ready(());
    }else{
      // Set waker so that the thread can wake up the current task
      // when the timer has completed, ensuring that the future is polled
      // again and sees that `completed = true`.
      //
      // It's tempting to do this once rather than repeatedly cloning
      // the waker each time. However, the `TimerFuture` can move between
      // tasks on the executor, which could cause a stale waker pointing
      // to the wrong task, preventing `TimerFuture` from waking up
      // correctly.
      //
      // N.B. it's possible to check for this using the `Waker::will_wake`
      // function, but we omit that here to keep things simple.
      share_state.waker=Some(ctx.waker().clone());
      Poll:Pending
    }
  }
}
```

Pretty simple, right? If the thread has set `shared_state.completed = true`, we're done! Otherwise, we clone the `Waker` for the current task and pass it to `shared_state.waker` so that the thread can wake the task back up.

Importantly, we have to update the `Waker` every time the future is polled because the future may have moved to a different task with a different `Waker`. This will happen when futures are passed around between tasks after being polled.

Finally, we need the API to actually construct the timer and start the thread:

```rust
impl TimerFuture{
  pub fn new (duration:Duration)->Self{
    let shared_state = Arc::new(Mutex::new(SharedState{
      completed:false,
      waker:None,,
    }));
    
    let thread_shared_state=shared_state.clone();
    thread::spawn(move||{
      thread::sleep(duration);
      let mut shared_state=thread_shared_state.lock().unwrap();
      shared_state.completed=true;
      if let Some(waker)=shared_state.waker.take(){
        waker.wake()
      }
    });
    TimerFuture{shared_state}
  }
}
```

Woot! That's all we need to build a simple timer future. Now, if only we had an executor to run the future on...

### 2.3 Applied: Build an Executor

Rust's `Future` are lazy: they won't do anything unless actively driven to completion. One way to drive a future to completion is to `.await` it inside an `async` function, but that just pushes the problem one level up: who will run the futures trturn from the top-level `async` functions? The answer is that we need a `Future` executor.

`Future` executors take a set of top-level `Future`s and run them to completion by calling `poll` whenever the `Future` can make progress. Typically, an executor will `poll` a future once to start off. When `Future`s indicate that they are ready to make progress by calling `wake()`, they are placed back onto a queue and `poll` is called again, repeating until the `Future` has completed.

In this section, we'll write our own simple executor capable of running a large number of top-level futures to completion concurrently.

For this example, we depend on the `futures` crate for the `ArcWake` trait, which provides an easy way to construct a `Waker`. Edit `Cargo.toml` to add a new dependency:

```toml
[package]
name = "timer_future"
version = "0.1.0"
authors = ["XYZ Author"]
edition = "2021"

[dependencies]
futures = "0.3"
```

Next, we need the following imports at the top of `src/main.rs`:

```rust
use{
  futures""{
    future::{BoxFuture, FutureExt},
    task::{waker_ref,ArcWake},
  },
  std::{
    future::Future,
    sync::mpsc::{sync_channel, Receiver, SyncSender},
    sync::{Arc,Mutex},
    task::{Context,Poll},
    time::Duration,
  },
  timer_future::TimerFuture,
};
```

Our executor will work by sending tasks to run over a channel. The executor will pull events off of the channel and run them. When a task is ready to do more work (is awoken ), it can schedule itself to be polled again by putting itself back onto the channel.

In this design, the executor itself just needs the receiving end of the task channel. The user will get sending end so that they can spawn new futures. Tasks themselves are just futures that can reschedule themselves, so we'll store them as a future paired with a sender that the task can use to require itself.

```rust
/// Task executor that receives tasks off of a channel and runs them.
struct Executor{
  ready_queue:Receiver<Arc<Task>>,
}

/// `Spawner` spawns new futures onto the task channel.
#[derive(Clone)]
struct Spawner{
  task_sender: SyncSender<Arc<Task>>,
}

/// A future that can reschedule itself to be polled by an `Executor`.
struct Task{
  /// In-progress future that should be pushed to completion.
  ///
  /// The `Mutex` is not necessary for correntness, since we only have
  /// one thread executing tasks at once. However, Rust isn't smart
  /// enough to know that `future` is only mutated from one thread,
  /// so we need to use the `Mutex` to prove thread-safety. A production
  ///  executor would not need this, and could use `UnsafeCell` instead.
  future: Mutex<Option<BoxFuture<'static, ()>>>,
  task_sender: SyncSender<Arc<Task>>,
}

fn new_executor_and_spawner()->(Executor,Spawner){
  // Maximum number of tasks to allow queueing in the channel at once.
  // This is just to make `sync_channel` happy, and wouldn't be present in
  // a real executor.
  const MAX_QUEUED_TASKS:usize 10_000;
  let(task_sender,ready_queue) = sync_channel(MAX_QUEUED_TASKS)；
  （Executor{ready_queue},Spawner{task_sender})
}
```

Let's also add a method to spawner to make it easy to spawn new futures. This method will take a future type, box it, and create a new `Arc<Task>` with it inside which can be enqueued onto the executor.

```rust
impl Spawner{
  fn spawn(&self, future: impl Future<Output=()> + 'static + Send){
    let future = future.boxed();
    let task = Arc::new(Task{
      future:Mutex::new(Some(future)),
      task_sender:self.task_sender.clone(),
    });
    self.task_sender.send(task).expect("too many tasks queued");
  }
}
```

To poll futures, we'll need to create a `Waker`. As discussed in the [task wakeups section](https://rust-lang.github.io/async-book/02_execution/03_wakeups.html), `Waker`s are responsible for scheduling a task to be polled again once `wake` is called. Remember that `Waker`s tell the executor exactly which task has become ready, allowing them to poll just the futures that are ready to make progress. The easiest way to create a new `Waker` is by implementing the `ArcWake` trait and then using the `waker_ref` or `.into_waker()` functions to turn an `Arc<impl ArcWake>` into a `Waker`. Let's implement `ArcWake` for our tasks to allow them to be turned into `Waker`s and awoken:

```rust
impl ArcWake for Task{
  fn wake_by_ref(arc_self: &Arc<Self>){
    // Implement `wake` by sending this task back onto the task channel
    // so that it will be polled again by the executor.
    let cloned = arc_self.clone();
    arc_self.task_sender.send(cloned).expect("too many tasks queued");
  }
}
```

When a `Waker` is created from an `Arc<Task>`, calling `wake()` on it will cause a copy of the `Arc` to be sent onto the task channel. Our executor then needs to pick up the task and poll it. Let's implement that:

```rust
impl Executor{
  fn run(&self){
    while let Ok(task) = self.ready_queue.recv(){
      // Take the future, and if it has not yet completed (is still Some),
      // poll it in an attempt to complete it.
      let mut future_slot = task.future.lock().unwrap();
      if let Some(mut future) = future_slot.take(){
        // Create a `LocalWaker` from the task itself
        let waker = waker_ref(&task);
        let context = &mut Context::from_waker(&*waker);
        // `BoxFuture<T>` is a type alias for
        // `Pin<Box<dyn Future<Output = T> + Send + 'static>>`.
        // We can get a `Pin<&mut dyn Future + Send + 'static>`
        // from it by calling the `Pin::as_mut` method.
        if future.as_mut().poll(context).is_pending(){
          // We're not done processing the future, so put it
          // back in its task to be run again in the future.
          *future_slot = Some(future);
        }
      }
    }
  }
}
```

Congratulations! We now have a working futures executor. We can even use it to run `async/.await` code and custom futures, such as the `TimerFuture` we wrote earlier:

```rust
fn main(){
  let (executor, spawner) = new_executor_and_spawner();
  
  // Spawn a task to print before and after waiting on a timer.
  spawner.spawn(async{
    println!("howdy!");
    // Wait for our timer future to complete after two seconds.
    TimerFuture::new(Duration::new(2,0)).await;
    println!("done!");
  });
  
  // Drop the spawner so that our executor knows it is finished and won't
  // receive more incoming tasks to run.
  drop(spawner);
  
  // Run the executor unitl the task queue is empty.
  // This will print "howdy!", pause, and then print "done!".
  executor.run();
}
```

### 2.4 Executors and System IO

In the previous section on [The Future Trait](https://rust-lang.github.io/async-book/02_execution/02_future.html), we discussed this example of a future that performed an asynchronous read on a socket:

```rust
pub struct SocketRead<'a>{
  socket: &'a Socket,
}

impl SimpleFuture for SocketRead<'_>{
  type Output = Vec<u8>;
  
  fn poll(&mut self, wake:fn()) -> Poll<Self::Output>{
    if self.socket.has_data_to_read(){
      // The socket has data -- read it into a buffer and return it.
      Poll::Ready(self.socket.read_buf())
    } else {
      // The socket does not yet have data.
      //
      // Arrange for `wake` to be called once data is available.
      // When data becomes available, `wake` will be called, and the 
      // user of this `Future` will know to call `poll` again and 
      // receive data.
      self.socket.set_readable_callback(wake);
      Poll::Pending
    }
  }
}
```

This future will read available data on a socket, and if no data is available, it will yield to the executor, requesting that its task be awoken when the socket becomes readable again. However, it's not clear from this example how the `Socket` type is implemented, and in particular it isn't obvious how the `set_readable_callback` function works. How can we arrange for `wake()` to be called once the socket becomes readable? One option would be to have athread that continually checks whether `socket` is readable, calling `wake()` when appropriate. However, this would be qiute inefficient, `socket` is readable, calling `wake()` when appropriate. However, this would be qiute inefficient, requiring a separate thread for each blocked IO future. This would greatly reduce the efficiency of our async code.

In practice, this problem is solved through integration with an IO-aware system blocking primitive, such as `epoll` on Linux, `kqueue` on FreeBSD and Mac OS, IOCP on Windows, and `port`s on Fuchsia (all of which are exposed through the cross-platform Rust crate `mio`). These primitives all allow a thread to block on pultiple asynchronous IO events, returning once on of the events completes. In practice, these APIs usually look something like:

```rust
struct IoBlocker{
  /* ... */
}

struct Event{
  // An ID uniquely identifying the event that occurred and was listened for.
  id:usize,
  // A set of signals to wait for, or which occurred.
  signals:Signals,
}

impl IoBlocker{
  /// Create a new collection of asynchronous IO events to block on.
  fn new()->Self{/* ... */}
  
  /// Express an interest in a particular IO event.
  fn add_io_event_interest(
    &self,
    
    /// The object on which the event will occur
    io_object: &IoObect,
    
    /// A set of signals that may appear on the `io_object` for
    /// which an event should be triggered, parired with
    /// an ID to give to events that result from this interest.
    event:Event,
  ){/* ... */}
}
  let mut io_blocker = IoBlocker::new();
  io_blocker.add_io_event_interest(&socket_1, Event{id:1,signals:READABLE},);
  io_blocker.add_io_event_interest(&socket_2, Event{id:2,signals:READABLE|WRITABLE},);
  
  let event = io_blocker.block();
  
  // prints e.g. "Socket 1 is now READABLE" if socket on became readable.
  println!("Socket {:?} is now {:?}",event.id, event.signals);
```

Futures executors can use these primitives to provide asynchronous IO objects such as sockets that can configure callbacks to be run when a particular IO event occurs. In the case of our `SocketRead` example above, the `Socket::set_readable_callback` function might look like the following pseudocode:

```rust
impl Socket{
  fn set_readable_callback(&self, waker:Waker)
{
  // `local_executor` is a reference to the local executor.
  // this could be provided at creation of the socket, but in practice
  // many executor implementations pass it down through thread local
  // storage for convenience.
  let local_executor = self.local_executor;

  // Unique ID for this IO object.
  let id = self.id;

  // Store the local waker in the executor's map so that it can be called
  // once the IO event arrives.
  local_executor.event_map.insert(id, waker);
  local_executor.add_io_event_interest(
    &self.socket_file_descriptor,
    Event { id, signals: READABLE },
  );
}}
```

We can now have just one executor thread which can receive and dispatch any IO event to the appropriate `Waker`, which will wake up the corresponding task, allowing the executor to deive more tasks to completion before returning to check for more IO events (and the cycle continues...).

## 3. async/ .await

In [the first chapter](https://rust-lang.github.io/async-book/01_getting_started/04_async_await_primer.html), we took a brief look at `async`/`.await`. This chapter will discuss `async`/`.await` in greater detail, explaining how it works and how `async` code differs from traditional Rust programs.

`async`/`.await` are special pieces of Rust syntax that make it possible to yield control of the current thread rather than blocking, allowing other code to make progress while waiting on an operation to complete.

There are two main ways to use `async`:`async fn` and `async` blocks. Each returns a value that implements the `Future` trait:

```rust
// `foo()` returns a type that implements `Future<Output = u8>`.
// `foo().await` will result in a value of type u8.
async fn foo()->u8{5}

fn bar()->impl Future<Output=u8> {
  async{
    // This `async` block results in a type that implements
    // `Future<Output = u8>`.
    let x:u8 = foo().await;
    x+5
  }
}
```

As we saw in the first chapter, `async` bodies and other futures are lazy: they do nothing until they are run. The most common way to run a `Future` is to `.await` it. When `.await` is called on a `Future`, it will attempt to run it to completion. If the `Future` is blocked, it will yield control of the current thread. When more progress can be made, the `Future` will be picked up by the executor and will resume running, allowing the `.await` to resolve.

**async Lifetimes**

Unlike traditional functions, `async fn` which take references or other non-`‘static` arguments return a `Future` which is bounded by the lifetime of the arguments:

```rust
// This function:
async fn foo(x: &u8)-> u8{*x}

// Is equivalent to this function:
fn foo_expanded<'a>(x:&'a u8)->impl Future<Output=u8>+'a{
  async move {*x}
}
```

This means that the future returned from an `async fn` must be `.await`ed while its non-`'static` argumutns are still valid. In tie common case of `.await`ing the future immediately after calling the function (as in `foo(&x).await`) this is not an issue. However, if storing the future or sending it over to another task or thread, this may be an issue.

One common workaround for turning an `async fn` with references-arguments into a `'static ` future is to boundle the argument with the call to the `async fn ` inside an `async` block:

```rust
fn bad()->impl Future<Output=u8>{
  let x=  5;
  borrow_x(&x) // ERROR:`x` does not live long enough
}

for good()->impl Future<Output=u8>{
  async{
    let x= 5;
    borrow_x(&x).await
  }
}
```

By moving the argument into the `async` block, we extend its life time to match that of the `Future` returned from the call to `good`.



**async move**

`async` blocks and dlosures allow the `move` keyword, much like normal closures. An `async move` block will take ownership of the variables it references, allowing it to out live the current scope, but giving up the ability to share those variables with other code:

```rust
/// `async` block:
///
/// Multiple different `async` blocks can access the same local variable
/// so long as they're executed within the variable's scope
async fn blocks(){
  let my_string = "foo".to_string();
  
  let future_one = async{
    //...
    println!("{}", my_string);
  }；
  
  let future_two = async{
    //...
    println!("{}", my_string);
  }；
  
  // Run both futures to completion, printing "foo"twice:
  let((),()) = futures::join!(future_one,future_two);
}

/// `async move` block:
///
/// Only one `async move` block can access the same captured variable, since
/// captures are moved into the `Future` generated by the `async move` block.
/// However, this allows the `Future` to outlive the original scope of the
/// variable:
fn move_block() -> impl Future<Output=()>{
  let my_string = "foo".to_string();
  async move{
      let future_one = async{
    //...
    println!("{}", my_string);
    }；

    let future_two = async{
      //...
      println!("{}", my_string);
    }；

    // Run both futures to completion, printing "foo"twice:
    let((),()) = futures::join!(future_one,future_two);
  }
}
```

**.awaiting on a Multithreaded Executor**

Note that, when using a multithreaded `Future` executor, a `Future` may move between threads, so any variables used in `async` bodies must be able to travel between threads, as any `.await` can potentially result in a switch to a new thread.

This means that it is not save to use `Rc`, `&RefCell` or any other types that don't implement the `Send` trait, including references to types that don't implement the `Sync	` trait.

(Caveat: it is possible to use these types as long as they aren't in scope during a call to `.await`)

Similarly, it isn't a good. Idea to hold a traditional non-futures-aware lock across an `.await`, as it can cause the thread pool to lock up: one task could take out a lock, `.await` and yield to the executor, allowing another task to attempt to take the lock and cause a deadlock. To avoid this, use the `Mutex` in `future::lock` rather than the one from `std::sync`.

## 4. Pinning

**Pinning**

To poll futures, they must be pinned using a special type called `Pin<T>`. If you read the explanation of [the `Future` trait](https://rust-lang.github.io/async-book/02_execution/02_future.html) in the previous section ["Executing `Future`s and Tasks"](https://rust-lang.github.io/async-book/02_execution/01_chapter.html), you'll recognize `Pin` from the `self: Pin<&mut Self>` in the `Future::poll` method's definition. But what does it mean, and why do we need it?

**Why Pinning**

`Pin` works in tandem with the `Unpin` marker. Pinning makes it possible to guarantee that an object implementing `!Unpin` won't ever be moved. To understand why this is necessary, we need to remember how `async`/`.await` works. Consider the following code:

```rust
let fut_one = /*...*/;
let fut_two = /*...*/;
async move{
  fut_one.await;
  fut_two.await;
}
```

Under the hood, this creaes an asynchronous type that implements `Future`, providing a `poll` method that looks something like this:

```rust
// The `Future` type generated by our `async {...}` block
struct AsyncFuture{
  f1: FutOne,
  f2:FutTwo,
  state:State,
}

// List of states our `async` block can be in
enum State{
  AwaitingFutOne,
  AwaitingFutTwo,
  Done,
}

impl Future for AsyncFuture{
  type Output=();
  fn poll(mut self:Pin<&mut Self>, cx: &mut Context<'_>)->Poll<()>{
    loop{
      match self.state{
        State::AwaitingFutOne=> match self.f1.poll(..){
          Poll::Ready(())=>self.state=State::AwaitingFutTwo,
          Poll::Pending=> return Poll:Pending, 
        },
        State::AwaitingFutTwo=> match self.f2.poll(..){
             Poll::Ready(())=>self.state=State::Done,
          Poll::Pending=> return Poll:Pending, 
        },
        State::Done=>{ return Poll:Ready(())},
      }
    }
  }
}
```

When `poll` is first called, it will poll `fut_one`. If `fut_one` can't complete, `AsyncFuture::poll` will return. Future calls to `poll` will pick up where the previous one left off. This process continues until the future is able to successfully complete.

However, what happens if we have an `async` block that uses references? For example:

```rust
async {
  let mut x = [0;128];
  let read_into_buf_fut=read_into_buf(&mut x);
  read_into_buf_fut.await;
  println!("{:?}",x);
}
```

What struct does this compile down to?

```rust
struct ReadIntoBuf<'a>{
  buf: &'a mut [u8], //points to `x` below
}

struct AsyncFuture{
  x: [u8,128],
  read_into_buf_fut:ReadIntoBuf<'what_lifetime?>,
}
```

Here the `ReadIntoBuf` future holds a reference into the other field of our structure, `x`. However, if `AsyncFuture` is moved, the location of `x` will move as well, invalidating the pointer stored in `read_into_buf_fut.buf`.

**Pinning in Detail**

Let's try to understand pinning by using an slightly simpler example. The problem we encounter above is a problem that ultimately boils down to how we handle references in self-referential types in Rust.

For now our example will look like this:

```rust
#[derive(Debug)]
strucy Test{
  a: String,
  b: *const String,
}

impl Test{
  fn new(txt: &str) ->Self{
    Self{
      a: String::from(txt),
      b: std::ptr::null(),
    }
  }
  
  fn init(&mut self){
    let self_ref:*const String=&self.a;
    self.b=self_ref;
  }
  
  fn a(&self)->&str{
    &self.a
  }
  
  fn b(&self)->&String{
    assert!(!self.b.is_null(),"Test::b called without Test::init being called first");
    unsafe{ &*(self.b)}
  }
}
```




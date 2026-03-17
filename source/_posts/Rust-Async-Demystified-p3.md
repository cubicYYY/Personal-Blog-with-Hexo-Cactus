---
title: 'Rust Async Demystified: Part 3 - Build Yourself a Minimal Runtime from Scratch'
tags:
  - Rust
  - Asynchronous Programming
  - Tutorial
  - Rust Async Demystified
categories:
  - Programming Languages
  - Rust
permalink: rust-async-demystified-p3/
date: 2026-03-16 00:41:05
---


> Author: CUBIC Y^3
> Feel free to share, but please credit the source and include a link to the original article. Thanks! :)

## Intro

In [Part 1](/rust-async-demystified-p1), we explored the fundamentals of asynchronous programming: what it is and why it matters. In [Part 2](/rust-async-demystified-p2), we broke down the core components of Rust's async system: Futures as state machines, executors driving them via `poll()`, Wakers for signaling readiness, and reactors that convert OS I/O events into task wakeups.

Now it's time to get our hands dirty: we're going to build a mini Rust async runtime from scratch.

By the end of this article, you will have a working async runtime called `fiber-runtime` that can:

1. **Execute futures** on one or many threads (the **executor**)
2. **Wake tasks** when a timer expires (a simple **leaf future**)
3. **Monitor sockets** for I/O readiness (the **reactor**)
4. **Serve concurrent TCP connections**, a fully async echo server

The entire runtime is ~350 lines of Rust (including comments). Let's begin.

---

## The Blueprint

Before writing any code, let's recall the architecture from [Part 2, Piece 5](/rust-async-demystified-p2#Putting-Executor-Waker-Reactor-Together). Our runtime has three collaborating components:

![architecture](architecture.svg)

- The **Spawner** wraps a future into a **Task** and pushes it into a channel.
- The **Executor** pulls tasks from the channel and calls `poll()`.
- If a future returns `Pending`, it will store a **Waker**, which will re-enqueue the task when the I/O becomes ready later.
- The **Reactor** runs on a background thread, monitoring I/O sources via the OS. When an event fires, it calls `waker.wake()`, which pushes the task back into the channel.

---

## Preparation

Create a new Rust project:

```bash
cargo new fiber-runtime --lib
cd fiber-runtime
```

Our dependencies in `Cargo.toml`:

```toml
[package]
name = "fiber-runtime"
version = "0.1.0"
edition = "2021"

[dependencies]
crossbeam-channel = "0.5"
futures = "0.3"
mio = { version = "1", features = ["os-poll", "net"] }
```

These are the dependencies our runtime needs.

- `crossbeam-channel`: A fast MPMC (multi-producer, multi-consumer) channel for the task queue. Basically, it's a queue with notification.
- `futures`: Provides `BoxFuture`, `FutureExt::boxed()`, and `ArcWake`. These are utilities we need for type-erasing and waking futures (we will explain this later). We could implement these by hand, but they're boilerplate.
- `mio`: A thin, cross-platform wrapper around OS I/O event notification (`epoll` on Linux, `kqueue` on macOS, `IOCP` on Windows). This is what Tokio, a widely used async runtime, uses under the hood. By using this, you don't need to bother with different I/O mechanisms across different platforms.

Our module structure:

```text
src/
  lib.rs            : module declarations
  executor.rs       : Task, Executor, Spawner, JoinHandle, block_on
  timer_future.rs   : a simple timer (thread-per-timer, for testing)
  reactor.rs        : the I/O event loop
  tcp.rs            : async TcpListener / TcpStream
```

```rust
// src/lib.rs
pub mod executor;
pub mod macros;
pub mod reactor;
pub mod tcp;
pub mod timer_future;

pub use executor::block_on;
```

Now let's build each piece for our async runtime.

---

## Phase 1: The Executor

This is the heart of the runtime -- the code that actually calls `poll()` on your futures.

### The Task

Recall from [Part 2, Piece 4](/rust-async-demystified-p2#Piece-4-Who-Calls-poll-The-Executor): the executor maintains a queue of **tasks**, where each task wraps a top-level future. We need to define what a task looks like:

```rust
use std::sync::{Arc, Mutex};
use crossbeam_channel::Sender;
use futures::future::BoxFuture;

struct Task {
    future: Mutex<Option<BoxFuture<'static, ()>>>,
    loopback_entrance: Sender<Arc<Task>>,
}
```

Let's unpack each field:

- **`future: Mutex<Option<BoxFuture<'static, ()>>>`**: The boxed, type-erased future this task is driving. It's wrapped in `Option` so we can `.take()` it out for polling (and put it back if `Pending`). The `Mutex` protects against concurrent access if multiple executor threads exist.

- **`loopback_entrance: Sender<Arc<Task>>`**: This is a clone of the task queue's sender channel. When the task is woken, it uses this channel to re-submit its own `Arc<Task>` back to the executor, ensuring it is re-queued and ready for polling.

Why `BoxFuture<'static, ()>`? All spawned tasks return `()`. If a user wants a typed return value (like `block_on` does), we wrap the original future in an adapter that sends the result through a separate channel. This keeps the task queue homogeneous -- no `Box<dyn Any>` or `downcast` needed.

### The Waker: `ArcWake`

In [Part 2, Piece 5](/rust-async-demystified-p2#Piece-5-How-Does-a-Future-Get-Woken-Up-The-Waker), we learned that `Waker` is the glue between futures and the executor. When something calls `waker.wake()`, the executor knows the task can make progress.

Rust's `std::task::Waker` is built from a raw vtable (`RawWakerVTable`), which is fiddly to implement by hand. The `futures` crate provides the `ArcWake` trait as a convenient shortcut: implement one method, and you get a `Waker` for free.

```rust
use futures::task::ArcWake;

impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        let cloned = arc_self.clone();
        let _ = arc_self.loopback_entrance.send(cloned);
    }
}
```

When `waker.wake()` is called, the task clones itself and sends the `Arc<Task>` back into the task queue. The executor will pick it up and re-poll the future later.¹ ²

### The Executor

Our executor is simple: a loop that pulls tasks from the channel and polls them.

```rust
use crossbeam_channel::Receiver;
use futures::task::waker_ref;
use std::task::Context;

#[derive(Clone)]
pub struct Executor {
    task_queue: Receiver<Arc<Task>>,
}

impl Executor {
    pub fn run(&self) {
        while let Ok(task) = self.task_queue.recv() {
            let mut future_slot = task.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                let waker = waker_ref(&task);
                let context = &mut Context::from_waker(&waker);
                if future.as_mut().poll(context).is_pending() {
                    // Still pending -- put the future back for next time.
                    *future_slot = Some(future);
                }
            }
        }
    }
}
```

Let's walk through the loop:

1. **`self.task_queue.recv()`**: Block until a task arrives. No busy-waiting, no wasted CPU.³

2. **`future_slot.take()`**: We take the future out of the `Option`. If it's `None`, another thread already took it (won't happen in single-threaded mode, but safe regardless).

3. **`waker_ref(&task)`**: Create a `Waker` backed by our `ArcWake` implementation, *without* cloning the `Arc`. This is more efficient than `waker()` which clones.

4. **`future.as_mut().poll(context)`**: Drive the state machine forward. As we learned in [Part 2, Piece 1](/rust-async-demystified-p2#A-Coroutine-Is-Just-a-State-Machine), each `poll()` advances the future across one `.await` point.

5. **If `Pending`**: put the future back in the slot. The future stored a clone of the `Waker` somewhere (inside the reactor, a timer thread, etc.), and that waker will eventually call `wake()`, re-enqueuing the task.

6. **If `Ready`**: the future is done. We don't put it back, so the `Option` stays `None`. The `Arc<Task>` is dropped at the end of this loop iteration, which drops the `loopback_entrance` sender. Once all senders are gone, the channel closes, and `recv()` returns `Err`, ending the loop.

Note that the `Executor` derives `Clone` -- we'll use this for [multi-threaded execution](#Going-Multi-Threaded) later.⁴

### The Spawner

The spawner is the producer side: it wraps a future into a `Task` and sends it into the channel.

```rust
use futures::future::FutureExt;
use std::future::Future;

#[derive(Clone)]
pub struct Spawner {
    queue_entrance: Sender<Arc<Task>>,
}

impl Spawner {
    pub fn spawn(&self, future: impl Future<Output = ()> + Send + 'static) {
        let task = Arc::new(Task {
            future: Mutex::new(Some(future.boxed())),
            loopback_entrance: self.queue_entrance.clone(),
        });
        self.queue_entrance
            .send(task)
            .expect("Executor has been dropped");
    }
}
```

`future.boxed()` (from `FutureExt`) converts the future into a `BoxFuture<'static, ()>`. This means the future is:

- **type-erased**: it hides its actual underlying type so all boxed futures look the same to the executor;
- **heap-allocated**: the future's data is stored on the heap rather than the stack; and
- **pinned**: the future is guaranteed not to move in memory, a requirement for safe async code.

This is necessary because the task queue can only store a single type (`Arc<Task>`), but each spawned future has a different concrete type (every `async` block produces a unique, anonymous type, see [Part 2, Piece 3](/rust-async-demystified-p2#Piece-3-What-async-fn-Actually-Compiles-To)).

Note the `Send + 'static` bounds -- the same constraint as `tokio::spawn()`.⁵

### Wiring Them Together

Now we connect the executor and spawner with a shared channel:

```rust
use crossbeam_channel::{bounded, unbounded};

pub fn new_executor_and_spawner(capacity: usize) -> (Executor, Spawner) {
    let (task_sender, task_receiver) = if capacity == 0 {
        unbounded()
    } else {
        bounded(capacity)
    };
    (
        Executor { task_queue: task_receiver },
        Spawner { queue_entrance: task_sender },
    )
}
```

We use `capacity = 0` as a sentinel for "unbounded" (not to be confused with crossbeam's `bounded(0)`, which is a rendezvous channel). An unbounded channel suits the general multi-task case. A bounded channel is useful for `block_on`, where we know there's exactly one task.

### `block_on`: Running a Single Future to Completion

`block_on` is the entry point users call from synchronous `main()`. It creates a mini executor, runs exactly one future, and returns the result:

```rust
// `async move { ... }` here is the async version of `move { ... }`.
// Just as `move { ... }` creates an anonymous closure that takes ownership of the variables it uses,
// `async move { ... }` creates an anonymous async block that captures (moves) its environment,
// and is compiled to a state-machine type similar to a normal async function (just without a name).
pub fn block_on<T: Send + 'static>(future: impl Future<Output = T> + Send + 'static) -> T {
    let (result_tx, result_rx) = bounded(1);
    let (ex, sp) = new_executor_and_spawner(1);
    sp.spawn(async move {
        let result = future.await;
        let _ = result_tx.send(result);
    });
    drop(sp);
    ex.run();
    result_rx.recv().expect("Future did not produce a result")
}
```

The trick here: the user's future might return any type `T`, but our task queue only stores `Future<Output = ()>`. So we **wrap** the user's future in an adapter that sends `T` through a typed crossbeam channel, then the outer future returns `()`. After the executor finishes, we receive the result from the channel. No `Box<dyn Any>`, no `downcast` -- just a typed channel.

The `drop(sp)` before `ex.run()` is essential for the executor to terminate.⁶

---

Here's the complete `executor.rs` at this point:

```rust
// src/executor.rs
use crossbeam_channel::{bounded, unbounded, Receiver, Sender};
use futures::{
    future::{BoxFuture, FutureExt},
    task::{waker_ref, ArcWake},
};
use std::{
    future::Future,
    sync::{Arc, Mutex},
    task::Context,
};

struct Task {
    future: Mutex<Option<BoxFuture<'static, ()>>>,
    loopback_entrance: Sender<Arc<Task>>,
}

impl ArcWake for Task {
    fn wake_by_ref(arc_self: &Arc<Self>) {
        let cloned = arc_self.clone();
        let _ = arc_self.loopback_entrance.send(cloned);
    }
}

#[derive(Clone)]
pub struct Executor {
    task_queue: Receiver<Arc<Task>>,
}

impl Executor {
    pub fn run(&self) {
        while let Ok(task) = self.task_queue.recv() {
            let mut future_slot = task.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                let waker = waker_ref(&task);
                let context = &mut Context::from_waker(&waker);
                if future.as_mut().poll(context).is_pending() {
                    *future_slot = Some(future);
                }
            }
        }
    }
}

#[derive(Clone)]
pub struct Spawner {
    queue_entrance: Sender<Arc<Task>>,
}

impl Spawner {
    pub fn spawn(&self, future: impl Future<Output = ()> + Send + 'static) {
        let task = Arc::new(Task {
            future: Mutex::new(Some(future.boxed())),
            loopback_entrance: self.queue_entrance.clone(),
        });
        self.queue_entrance
            .send(task)
            .expect("Executor has been dropped");
    }
}

pub fn new_executor_and_spawner(capacity: usize) -> (Executor, Spawner) {
    let (task_sender, task_receiver) = if capacity == 0 {
        unbounded()
    } else {
        bounded(capacity)
    };
    (
        Executor { task_queue: task_receiver },
        Spawner { queue_entrance: task_sender },
    )
}

pub fn block_on<T: Send + 'static>(future: impl Future<Output = T> + Send + 'static) -> T {
    let (result_tx, result_rx) = bounded(1);
    let (ex, sp) = new_executor_and_spawner(1);
    sp.spawn(async move {
        let result = future.await;
        let _ = result_tx.send(result);
    });
    drop(sp);
    ex.run();
    result_rx.recv().expect("Future did not produce a result")
}
```

That's a complete, multi-threaded async executor in 90 lines. But we can't test it yet -- we need a future that actually does something asynchronous.

---

## Phase 2: A Simple `TimerFuture`

### The Simplest Leaf Future

To test our executor, we need a **leaf future**: a future that talks to something external and uses the `Waker` to signal completion. The simplest possible version: spawn an OS thread that sleeps for a duration, then wakes the task.

This is adapted from the [Rust Async Book](https://rust-lang.github.io/async-book/02_execution/03_wakeups.html):

```rust
// src/timer_future.rs
use std::{
    future::Future,
    pin::Pin,
    sync::{Arc, Mutex},
    task::{Context, Poll, Waker},
    thread,
    time::Duration,
};

pub struct TimerFuture {
    shared_state: Arc<Mutex<SharedState>>,
}

struct SharedState {
    completed: bool,
    waker: Option<Waker>,
}

impl Future for TimerFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        let mut shared_state = self.shared_state.lock().unwrap();
        if shared_state.completed {
            Poll::Ready(())
        } else {
            shared_state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}

impl TimerFuture {
    pub fn new(duration: Duration) -> Self {
        let shared_state = Arc::new(Mutex::new(SharedState {
            completed: false,
            waker: None,
        }));

        let thread_shared_state = shared_state.clone();
        thread::spawn(move || {
            thread::sleep(duration);
            let mut shared_state = thread_shared_state.lock().unwrap();
            shared_state.completed = true;
            if let Some(waker) = shared_state.waker.take() {
                waker.wake()
            }
        });

        TimerFuture { shared_state }
    }
}
```

This demonstrates the **`Waker` contract** from [Part 2, Piece 5](/rust-async-demystified-p2#The-Wake-up-Contract):

1. The executor calls `poll()`. The timer hasn't elapsed, so we **clone and store the waker** (`cx.waker().clone()`), and return `Pending`.
2. A background thread sleeps for the given duration. When it wakes, it sets `completed = true` and calls `waker.wake()`.
3. `waker.wake()` triggers our `ArcWake` implementation, which sends the task back into the channel.
4. The executor re-polls. This time `completed` is `true`, so we return `Ready(())`.

### Testing It

Let's see if it works:

```rust
// examples/sleep.rs
use std::time::Duration;
use fiber_runtime::{executor::new_executor_and_spawner, timer_future::TimerFuture};

fn main() {
    let (executor, spawner) = new_executor_and_spawner(0);

    for i in 0..4 {
        spawner.spawn(async move {
            println!("Task {i}: waiting {} seconds...", 5 - i);
            TimerFuture::new(Duration::from_secs(5 - i)).await;
            println!("Task {i}: done!");
        });
    }

    drop(spawner);
    executor.run();
}
```

```bash
$ cargo run --example sleep
Task 0: waiting 5 seconds...
Task 1: waiting 4 seconds...
Task 2: waiting 3 seconds...
Task 3: waiting 2 seconds...
Task 3: done!
Task 2: done!
Task 1: done!
Task 0: done!
```

All four tasks start immediately (they all print their "waiting" message), then complete in order of their timeout duration -- shortest first. **That's concurrency.** Four tasks interleaving on a single thread, driven by wakers from background timer threads.

### Going Multi-Threaded

Remember that our `Executor` is `Clone`. Let's run it on multiple threads:

```rust
// examples/racing.rs
use std::thread::{self, available_parallelism, JoinHandle};
use std::time::Duration;
use fiber_runtime::{executor::new_executor_and_spawner, timer_future::TimerFuture};

fn main() {
    let (executor, spawner) = new_executor_and_spawner(0);

    for i in 0..512 {
        spawner.spawn(async move {
            TimerFuture::new(Duration::from_secs(0)).await;
            println!("{} done by {:?}", i, thread::current().id());
        });
    }

    drop(spawner);

    let mut threads: Vec<JoinHandle<_>> = vec![];
    for _ in 0..available_parallelism().unwrap().get() {
        let cloned = executor.clone();
        threads.push(thread::spawn(move || {
            cloned.run();
        }));
    }

    for handle in threads {
        handle.join().expect("Thread panicked!");
    }
}
```

512 tasks are distributed across all available CPU cores. Each thread runs a clone of the executor, pulling from the same shared channel. Crossbeam handles the synchronization.

You'll see tasks completed by different `ThreadId`s in the output, confirming true parallelism.⁷

---

## Phase 3: The Reactor

### The Problem with `TimerFuture`

Our `TimerFuture` works, but it has a fundamental flaw: **it spawns one OS thread per timer**. If you create 10,000 timers, that's 10,000 OS threads -- each consuming stack memory and scheduler overhead. This defeats the purpose of async programming.

Real-world async workloads are often I/O-bound: reading from sockets, accepting connections, waiting for network responses. The key insight from [Part 2, Piece 5](/rust-async-demystified-p2#The-Reactor-Connecting-to-the-OS) is that the OS already has efficient mechanisms to monitor many I/O sources simultaneously (e.g., `epoll` on Linux). A single thread can monitor multiple sockets.

The component that interfaces with these OS mechanisms is the **reactor**.

### Designing the Reactor

Our reactor will:

1. Run a **single background thread** with an event loop.
2. Use `mio::Poll` to call `epoll_wait` / `kevent` / `GetQueuedCompletionStatus` under the hood.
3. Maintain a `HashMap<Token, Waker>` -- mapping each I/O source to the waker of the task waiting on it.
4. When the OS reports an event, look up the waker and call `waker.wake()`.

It's a **global singleton** (initialized lazily via `OnceLock`) so any future in the program can register with it.

```rust
// src/reactor.rs
use mio::{Events, Interest, Poll, Token};
use std::{
    collections::HashMap,
    future::Future,
    io,
    pin::Pin,
    sync::{
        atomic::{AtomicUsize, Ordering},
        Arc, Mutex, OnceLock,
    },
    task::{Context, Poll as TaskPoll, Waker},
    thread,
};

static REACTOR: OnceLock<Reactor> = OnceLock::new();

pub struct Reactor {
    registry: mio::Registry,
    wakers: Arc<Mutex<HashMap<Token, Waker>>>,
    next_token: AtomicUsize,
}
```

The fields:

- **`registry`**: a handle cloned from `mio::Poll`. The `Poll` itself lives on the background thread; the `Registry` lets us register/deregister I/O sources from any thread.
- **`wakers`**: shared between the reactor struct (where futures store wakers) and the background thread (where wakers are called). An `Arc<Mutex<HashMap>>` is the simplest correct approach.
- **`Token`**: A `Token` is a small numeric identifier used by the reactor to distinguish between different I/O sources (like sockets or timers). When an event occurs, the OS reports the corresponding token, allowing the reactor to quickly look up which task's waker to notify. The `next_token` field is an atomic counter that ensures each new I/O source is assigned a unique token.

### Initialization and the Event Loop

```rust
impl Reactor {
    pub fn get() -> &'static Reactor {
        REACTOR.get_or_init(|| {
            let poll = Poll::new().expect("failed to create mio::Poll");
            let registry = poll
                .registry()
                .try_clone()
                .expect("failed to clone mio registry");
            let wakers: Arc<Mutex<HashMap<Token, Waker>>> =
                Arc::new(Mutex::new(HashMap::new()));

            let wakers_clone = wakers.clone();
            thread::Builder::new()
                .name("reactor".into())
                .spawn(move || Self::event_loop(poll, wakers_clone))
                .expect("failed to spawn reactor thread");

            Reactor {
                registry,
                wakers,
                next_token: AtomicUsize::new(0),
            }
        })
    }
```

On first call, `get()`:

1. Creates a `mio::Poll` instance (this calls `epoll_create` / `kqueue()` / `CreateIoCompletionPort` under the hood).
2. Clones the `Registry` so we can register sources from any thread.
3. Spawns the background reactor thread with the `Poll` and a clone of the waker map.
4. Returns the `Reactor` struct, which stores the registry and the waker map.

The event loop itself is simple:

```rust
    fn event_loop(mut poll: Poll, wakers: Arc<Mutex<HashMap<Token, Waker>>>) {
        let mut events = Events::with_capacity(64);
        loop {
            // Block until the OS reports at least one readiness event.
            poll.poll(&mut events, None).expect("reactor: mio poll failed");

            let mut map = wakers.lock().unwrap();
            for event in events.iter() {
                if let Some(waker) = map.remove(&event.token()) {
                    waker.wake();
                }
            }
        }
    }
```

`poll.poll(&mut events, None)` is the core: it blocks the reactor thread (efficiently, at the OS level) until one or more registered I/O sources have readiness events (so blocking releases CPU resources). For each event, we look up the token in our waker map to find the corresponding `waker`, then call `waker.wake()`. This re-enqueues the corresponding task in the executor's channel. The task will be re-polled, retry its I/O operation, and (hopefully) succeed this time.

Compare this with `TimerFuture`'s approach: instead of having one OS thread per timer calling `wake()`, we have **one reactor thread** listening for *all* I/O sources and calling `wake()` when any of them is ready. That's the whole point: we don't need 100 threads for 100 I/O tasks; a single reactor thread is capable of handling them all.

### Helper Methods

The rest of the reactor is just plumbing:

```rust
    pub fn token(&self) -> Token {
        Token(self.next_token.fetch_add(1, Ordering::Relaxed))
    }

    pub fn register(
        &self,
        source: &mut impl mio::event::Source,
        token: Token,
        interest: Interest,
    ) -> io::Result<()> {
        self.registry.register(source, token, interest)
    }

    pub fn deregister(&self, source: &mut impl mio::event::Source) -> io::Result<()> {
        self.registry.deregister(source)
    }

    pub fn set_waker(&self, token: Token, waker: Waker) {
        self.wakers.lock().unwrap().insert(token, waker);
    }

    pub fn remove_waker(&self, token: Token) {
        self.wakers.lock().unwrap().remove(&token);
    }
}
```

- `token()`: allocate a unique token for a new I/O source.
- `register()` / `deregister()`: forward to `mio::Registry` to add/remove sources from the OS poller.
- `set_waker()` / `remove_waker()`: manage the `Token -> Waker` mapping.

### `WaitReady`: The Bridge Future

We need one more piece: a small future that async code can `.await` to yield until the reactor fires an event for a given token.

```rust
pub(crate) struct WaitReady {
    token: Token,
    submitted: bool,
}

impl WaitReady {
    pub fn new(token: Token) -> Self {
        Self { token, submitted: false }
    }
}

impl Future for WaitReady {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> TaskPoll<()> {
        if self.submitted {
            // We were woken -- the I/O event fired (or a spurious wake).
            // The caller will retry the I/O operation in a loop.
            TaskPoll::Ready(())
        } else {
            // First poll: hand our waker to the reactor.
            Reactor::get().set_waker(self.token, cx.waker().clone());
            self.submitted = true;
            TaskPoll::Pending
        }
    }
}
```

This future is **one-shot**: on the initial poll, it registers the waker with the reactor and yields `Pending`. When the reactor fires, a subsequent poll returns `Ready(())`. In practice, async TCP methods loop around this future, retrying I/O operations as each readiness notification arrives. We'll see this pattern later.

> NOTE: The caller wraps this in a loop because spurious wakes are always possible.⁸

---

## Phase 4: Async TCP

Now that we have an executor and a reactor, making TCP asynchronous requires wrapping TCP primitives in our own infrastructure. Every async runtime needs custom I/O types to interact with runtime-specific components like the reactor.

### The Pattern: Try, Then Register

Every async I/O method follows the same two-step pattern:

```text
loop {
    1. Try the non-blocking syscall (read, write, accept).
    2. If it succeeds → return the result.
       If WouldBlock → WaitReady::new(token).await, then retry.
       If error      → return the error.
}
```

This is the exact flow described in [Part 2, Piece 5](/rust-async-demystified-p2#Putting-Executor-Waker-Reactor-Together). Let's implement it.

### `TcpListener`

```rust
// src/tcp.rs
use crate::reactor::{Reactor, WaitReady};
use mio::Interest;
use std::io::{self, Read, Write};
use std::net::SocketAddr;

pub struct TcpListener {
    inner: mio::net::TcpListener,
    token: mio::Token,
}

impl TcpListener {
    pub fn bind(addr: SocketAddr) -> io::Result<Self> {
        let mut inner = mio::net::TcpListener::bind(addr)?;
        let reactor = Reactor::get();
        let token = reactor.token();
        reactor.register(&mut inner, token, Interest::READABLE)?;
        Ok(Self { inner, token })
    }

    pub async fn accept(&self) -> io::Result<(TcpStream, SocketAddr)> {
        loop {
            match self.inner.accept() {
                Ok((stream, addr)) => return Ok((TcpStream::from_mio(stream)?, addr)),
                Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
                    WaitReady::new(self.token).await;
                }
                Err(e) => return Err(e),
            }
        }
    }
}

impl Drop for TcpListener {
    fn drop(&mut self) {
        let _ = Reactor::get().deregister(&mut self.inner);
        Reactor::get().remove_waker(self.token);
    }
}
```

In `bind()`:

- `mio::net::TcpListener::bind(addr)` creates a **non-blocking** listening socket (mio sets `O_NONBLOCK` automatically).
- We register it with the reactor for `READABLE` interest, i.e. "wake me when a client tries to connect."

In `accept()`:

- Try the non-blocking `accept()` syscall.
- If a client is waiting, we get a `TcpStream` back immediately.
- If `WouldBlock` (no client yet), we await `WaitReady`. The reactor will wake us when a connection arrives.
- On wake, we loop back and retry. This time, `accept()` succeeds.

The `Drop` impl deregisters from the reactor when the listener is dropped, preventing stale events.

### `TcpStream`

```rust
pub struct TcpStream {
    inner: mio::net::TcpStream,
    token: mio::Token,
}

impl TcpStream {
    fn from_mio(mut stream: mio::net::TcpStream) -> io::Result<Self> {
        let reactor = Reactor::get();
        let token = reactor.token();
        reactor.register(&mut stream, token, Interest::READABLE | Interest::WRITABLE)?;
        Ok(Self { inner: stream, token })
    }

    pub async fn read(&self, buf: &mut [u8]) -> io::Result<usize> {
        loop {
            let mut stream_ref = &self.inner;
            match stream_ref.read(buf) {
                Ok(n) => return Ok(n),
                Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
                    WaitReady::new(self.token).await;
                }
                Err(e) => return Err(e),
            }
        }
    }

    pub async fn write(&self, buf: &[u8]) -> io::Result<usize> {
        loop {
            let mut stream_ref = &self.inner;
            match stream_ref.write(buf) {
                Ok(n) => return Ok(n),
                Err(e) if e.kind() == io::ErrorKind::WouldBlock => {
                    WaitReady::new(self.token).await;
                }
                Err(e) => return Err(e),
            }
        }
    }

    pub async fn write_all(&self, buf: &[u8]) -> io::Result<()> {
        let mut written = 0;
        while written < buf.len() {
            written += self.write(&buf[written..]).await?;
        }
        Ok(())
    }
}

impl Drop for TcpStream {
    fn drop(&mut self) {
        let _ = Reactor::get().deregister(&mut self.inner);
        Reactor::get().remove_waker(self.token);
    }
}
```

The `read()` and `write()` methods follow the exact same try-then-register pattern. `write_all()` is a convenience that loops `write()` until the entire buffer is sent (handling partial writes).

The stream is registered with `READABLE | WRITABLE` interest, sharing a single token and waker slot.⁹ ¹⁰

---

## Phase 5: The Grand Finale: Our Own TCP Echo Server

Let's bring all the components together with a highly efficient TCP echo server. This server concurrently handles multiple connections, all on a single-threaded executor:

```rust
// examples/echo_server.rs
use fiber_runtime::{
    executor::new_executor_and_spawner,
    tcp::TcpListener,
};

fn main() {
    let (executor, spawner) = new_executor_and_spawner(0);

    let sp = spawner.clone();
    spawner.spawn(async move {
        let addr = "127.0.0.1:8080".parse().unwrap();
        let listener = TcpListener::bind(addr).unwrap();
        println!("Echo server listening on {addr}");
        println!("Test with: nc 127.0.0.1 8080");

        loop {
            let (stream, peer) = listener.accept().await.unwrap();
            println!("[{peer}] connected");

            // Spawn a new task for each connection.
            sp.spawn(async move {
                let mut buf = [0u8; 1024];
                loop {
                    let n = match stream.read(&mut buf).await {
                        Ok(0) | Err(_) => break, // EOF or error
                        Ok(n) => n,
                    };
                    if stream.write_all(&buf[..n]).await.is_err() {
                        break;
                    }
                }
                println!("[{peer}] disconnected");
            });
        }
    });

    drop(spawner);
    executor.run();
}
```

Let's try it:

```bash
$ cargo run --example echo_server
Echo server listening on 127.0.0.1:8080
```

In another terminal:

```bash
$ nc 127.0.0.1 8080
hello
hello
world
world
```

(If the port is occupied by another process, just modify the port number.)

### Echo Server: What's Happening Under the Hood

Let's trace the full lifecycle of a connection through our runtime:

**1. Accept loop is polling**
The accept task called `listener.accept()`, which returned `WouldBlock`. `WaitReady` stored the waker in the reactor. The task is parked, and the executor thread is idle.

**2. Client connects**
The OS signals the reactor thread: "the listening socket is readable." The reactor's `mio::Poll::poll()` returns. The reactor looks up the token, finds the waker, calls `waker.wake()`. The accept task is re-enqueued in the channel.

**3. Executor re-polls the accept task**
The accept loop retries `listener.accept()` -- this time it succeeds, returning a `TcpStream`. We spawn a new echo task for this connection and loop back to waiting for the next client.

**4. Echo task starts**
The echo task calls `stream.read(&mut buf)`, which returns `WouldBlock` (no data yet). `WaitReady` stores the waker. The task is parked.

**5. Client sends data**
The OS signals the reactor: "the stream socket is readable." The reactor wakes the echo task. The executor re-polls it. `stream.read()` succeeds, returning the data. `stream.write_all()` echoes it back (possibly with a reactor round-trip if the send buffer is full).

**6. Client disconnects**
`stream.read()` returns `Ok(0)` (EOF). The echo task breaks out of the loop, prints the disconnect message, and completes. The `TcpStream` is dropped, which deregisters from the reactor.

All of this happens on two threads: the executor thread (polling tasks) and the reactor thread (monitoring sockets).

This is the entire point of async programming, realized in ~300 lines of Rust.

---

## Phase 6: Bonus -- `JoinHandle` (Getting Results from Spawned Tasks)

### The Problem

So far, `spawn()` is fire-and-forget: you hand it a future that returns `()`, and that's it. But what if you want to spawn a computation and later `.await` its result?

```rust
let handle = spawner.spawn_with_handle(async { 1 + 1 });
let result = handle.await; // 2
```

This is exactly what Tokio's `JoinHandle` does. Let's add it to our runtime.

### Shared State: A Familiar Pattern

Remember `TimerFuture`? It used a shared `Arc<Mutex<SharedState>>` with a `completed` flag and a `waker` slot. `JoinHandle` uses the *exact same pattern*:

```rust
use std::{
    pin::Pin,
    task::{Context, Poll, Waker},
};

struct JoinState<T> {
    result: Option<T>,
    waker: Option<Waker>,
}

pub struct JoinHandle<T> {
    state: Arc<Mutex<JoinState<T>>>,
}
```

Two sides share the `JoinState`:

- The **producer** (the spawned task) writes `result` and wakes the waker.
- The **consumer** (`JoinHandle`) stores the waker and reads the result.

### Implementing `Future` for `JoinHandle`

```rust
impl<T: Send + 'static> Future for JoinHandle<T> {
    type Output = T;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<T> {
        let mut state = self.state.lock().unwrap();
        if let Some(result) = state.result.take() {
            Poll::Ready(result)
        } else {
            state.waker = Some(cx.waker().clone());
            Poll::Pending
        }
    }
}
```

This follows the standard waker contract: check if the result is ready; if not, store the waker and return `Pending`. When the spawned task completes, it will call `waker.wake()`, and the executor will re-poll us, obtaining the result.

### Refactoring `Spawner`

We extract the task-creation logic into a private `spawn_inner()` method, then build `spawn_with_handle()` on top:

```rust
impl Spawner {
    pub fn spawn(&self, future: impl Future<Output = ()> + Send + 'static) {
        self.spawn_inner(future.boxed());
    }

    pub fn spawn_with_handle<T: Send + 'static>(
        &self,
        future: impl Future<Output = T> + Send + 'static,
    ) -> JoinHandle<T> {
        let state = Arc::new(Mutex::new(JoinState {
            result: None,
            waker: None,
        }));
        let state_clone = state.clone();
        self.spawn_inner(
            async move {
                let result = future.await;
                let mut s = state_clone.lock().unwrap();
                s.result = Some(result);
                if let Some(waker) = s.waker.take() {
                    waker.wake();
                }
            }
            .boxed(),
        );
        JoinHandle { state }
    }

    fn spawn_inner(&self, future: BoxFuture<'static, ()>) {
        let task = Arc::new(Task {
            future: Mutex::new(Some(future)),
            loopback_entrance: self.queue_entrance.clone(),
        });
        self.queue_entrance
            .send(task)
            .expect("Executor has been dropped");
    }
}
```

The key trick: `spawn_with_handle()` wraps the user's `Future<Output = T>` using a new `async move { ... }` block (the async block passed to `.boxed()`). This async block acts as a **wrapper future**: it awaits the inner future (`future.await`), stores the result into the shared `JoinState`, and wakes any waiting task if a waker is present.

How does `.await` on a `JoinHandle` retrieve the result? When you `.await` a `JoinHandle`, its `Future` implementation checks if the result is available in the `JoinState`. If not, it stores the current waker there, so the wrapper future can wake the waiting task when the result is ready. On the next poll, once the result is present, the `JoinHandle` yields it as the output of `.await`.

This wrapper future always returns `()`, letting it fit into the homogeneous task queue, while the actual `T` result is delivered back to the awaiting task through the `JoinState`, waking and providing the awaited value at the right time.

Here is the full picture:

![Future with Return Value: the workflow](future-with-return-value.svg)

1. **Allocate shared state**: `spawn_with_handle` creates the `JoinState` with `result: None, waker: None`, wrapped in `Arc<Mutex<>>` (`let state = Arc::new(Mutex::new(JoinState { ... }))`). Two `Arc` clones are made.
2. **Enqueue the wrapper task**: one `Arc` clone (`state_clone`) is moved into the `async move { ... }` wrapper, which is boxed and sent into the task queue via `spawn_inner` (`self.spawn_inner( async move { ... }.boxed() )`).
3. **Return the handle**: the other `Arc` clone goes into the `JoinHandle` returned to the parent task (`JoinHandle { state }`). At this point the wrapper is sitting in the queue, not yet polled.
4. **Parent hits `.await`**: the parent task is already being polled by the executor. It continues past `spawn_with_handle` and reaches `handle.await`, which polls the `JoinHandle`'s `Future` impl. It locks the `JoinState`, sees `result` is `None` (`if let Some(result) = state.result.take()` -- falls to the `else` branch), stores the parent's waker (`state.waker = Some(cx.waker().clone())`), and returns `Pending`. The parent task is now parked.
5. **Executor picks up the wrapper**: the executor's `run()` loop pulls the Wrapper Task from the queue and polls it. Inside the wrapper, `future.await` (`let result = future.await`) drives the user's future to completion, producing value `T`.
6. **Store result**: the wrapper locks the `JoinState` and writes the value (`s.result = Some(result)`).
7. **Wake the parent**: the wrapper takes the stored waker and calls `waker.wake()` (`if let Some(waker) = s.waker.take() { waker.wake() }`). This re-enqueues the parent task into the executor's channel. The wrapper future is now done and gets dropped.
8. **Parent re-polled**: the executor picks up the parent task again and re-polls the `JoinHandle`. This time `state.result.take()` returns `Some(T)`, so it returns `Poll::Ready(result)`. The `.await` resolves and the parent has the value.

Note that the ordering of steps 4 and 5 doesn't have to be this way: if the wrapper finishes *before* the parent awaits, the result is already in `JoinState.result` and step 4 returns `Ready` immediately (the `if let Some(result)` branch). The `JoinState` acts as a rendezvous point -- the same `Arc<Mutex<>>` pattern as `TimerFuture`, just carrying a typed value instead of a boolean flag.

### Trying It Out

```rust
// examples/join_handle.rs
use fiber_runtime::executor::new_executor_and_spawner;
use fiber_runtime::timer_future::TimerFuture;
use std::time::Duration;

fn main() {
    let (executor, spawner) = new_executor_and_spawner(0);

    let sp = spawner.clone();
    spawner.spawn(async move {
        let h1 = sp.spawn_with_handle(async {
            TimerFuture::new(Duration::from_secs(1)).await;
            println!("task 1 done");
            10
        });
        let h2 = sp.spawn_with_handle(async {
            TimerFuture::new(Duration::from_secs(2)).await;
            println!("task 2 done");
            32
        });

        let sum = h1.await + h2.await;
        println!("sum = {sum}");
        assert_eq!(sum, 42);
    });

    drop(spawner);
    executor.run();
}
```

```bash
$ cargo run --example join_handle
task 1 done
task 2 done
sum = 42
```

Both tasks run concurrently (task 1 finishes first since its timer is shorter), and their results flow back through the `JoinHandle`s. Note that we need `let sp = spawner.clone()` because the `async move` block takes ownership of `sp`, while we still need `spawner` alive to call `spawner.spawn()` and later `drop(spawner)`.

---

## Recap: What We Built

Let's map our implementation back to the pieces from [Part 2](/rust-async-demystified-p2):

| Part 2 Concept | Our Implementation | Lines |
| - | - | - |
| **Piece 1**: Coroutines as state machines | The compiler does this for us via `async`/`await` | 0 |
| **Piece 2**: The `Future` trait | `TimerFuture`, `WaitReady`, and `JoinHandle` implement `Future` by hand; `async fn` generates the rest | ~100 |
| **Piece 3**: What `async fn` compiles to | Every `async move { ... }` block in our code | 0 |
| **Piece 4**: The Executor | `Executor::run()`, `Spawner::spawn()`, `spawn_with_handle()`, `block_on()` | ~90 |
| **Piece 5**: Waker + Reactor | `ArcWake for Task`, `Reactor`, `WaitReady` | ~100 |
| **Piece 6**: `Pin` | Handled by `BoxFuture` (which is `Pin<Box<dyn Future>>`) and the compiler | 0 |

The components we wrote by hand total ~350 lines of logic. The compiler handled the state machine generation (Piece 1, 3) and pinning (Piece 6) for free. That's the power of Rust's async design: the *interface* (`Future`, `Waker`, `Poll`) is tiny, and the compiler does the heavy lifting.

### What a Production Runtime Adds

Our runtime is designed for educational purposes. The following features are implemented by production-ready frameworks like Tokio, async-std, and smol in addition to what we've covered:

- **Timer wheel**: Instead of one thread per timer (`TimerFuture`) or one mio registration per timer, a single data structure (a hierarchical timing wheel) manages millions of timers efficiently.
- **Work-stealing scheduler**: Instead of a shared MPMC channel, each worker thread has a local queue with stealing from other threads' queues. This improves cache locality.
- **I/O driver integration**: The reactor and executor are fused -- the executor thread also runs `mio::Poll::poll()` when idle, eliminating the dedicated reactor thread.
- **Cancellation**: Our `JoinHandle` lets you `.await` a result, but production runtimes add *cancellation* -- dropping a `JoinHandle` can cancel the spawned task, freeing resources immediately.
- **`AsyncRead` / `AsyncWrite` traits**: Standardized traits for async I/O, analogous to `std::io::Read` / `Write`.
- **Cooperative yielding**: If a future does too much CPU work in one `poll()`, the runtime can preempt it to keep other tasks responsive.
- ...and more.

But the core architecture is similar: **task queue + waker + reactor**.

---

## What We Learned

Through the three parts of this series, we've journeyed from "what is async?" all the way to a working runtime:

1. **[Part 1](/rust-async-demystified-p1)**: Async programming is a way to achieve concurrency by switching between tasks when waiting for I/O, instead of blocking. It's orthogonal to parallelism.

2. **[Part 2](/rust-async-demystified-p2)**: Rust's async design is built on stackless coroutines compiled into state machines, driven by a pull-based polling model. The `Future` trait, the `Waker` mechanism, and the executor/reactor architecture work together to make this efficient.

3. **Part 3 (this article)**: We built it all from scratch. The executor (with `JoinHandle`) is a channel-pulling loop. The reactor is a `mio::Poll` event loop on a background thread. Async TCP demonstrates how the infrastructure is designed and used.

The magic of async Rust isn't magic at all. It's a beautifully composable system where each layer does one simple thing:

- **`async fn`** → the compiler generates a state machine.
- **`poll()`** → the executor asks "can you make progress?"
- **`Waker`** → the future says "I'll tell you when I can."
- **The reactor** → the OS says "this socket is ready."
- **The channel** → the waker says "put this task back in the queue."

Each piece is small enough to understand in isolation, and they compose into a system that can serve thousands of concurrent connections on a single thread. That's the zero-cost abstraction Rust promises -- and now you've seen every line of code that makes it happen.

The complete source code is available at: [github.com/CubicYYY/fiber-runtime](https://github.com/CubicYYY/fiber-runtime)

---

## References

[Asynchronous Programming in Rust (The Async Book)](https://rust-lang.github.io/async-book/)
[The Rust `Future` trait](https://doc.rust-lang.org/std/future/trait.Future.html)
[crossbeam-channel](https://docs.rs/crossbeam-channel/)
[mio - Metal I/O](https://docs.rs/mio/)
[futures crate - ArcWake](https://docs.rs/futures/latest/futures/task/trait.ArcWake.html)
[Tokio Tutorial](https://tokio.rs/tokio/tutorial)
[The Scoped Task Trilemma - without.boats](https://without.boats/blog/the-scoped-task-trilemma/)

Figures are created using [Excalidraw](https://excalidraw.com/).

---

## Notes

¹ The `let _ =` silently ignores send errors. This is intentional: during shutdown the receiver side of the channel is dropped and `send()` returns `Err`. That's normal -- the task was going to be dropped anyway.

² Modern runtimes (e.g., Tokio) use a more optimized waker based on `waker_ref`, which avoids the `Arc::clone()` overhead. The `futures` crate provides `waker_ref()` for exactly this, and our executor uses it when constructing the `Context`. But the *wake* path still needs to clone the `Arc` to enqueue the task, because the channel needs an owned value.

³ Crossbeam's `recv()` is clever: it spins briefly (to catch tasks that arrive quickly), then falls back to `thread::park()`, which on Linux uses a futex (fast userspace mutex) to put the thread to sleep until it's woken up. A futex allows the OS to efficiently suspend and resume threads, minimizing CPU usage while waiting. This strategy makes `recv()` efficient for both quick bursts of work and long idle periods.

⁴ The `Executor` is cloneable because crossbeam's `Receiver` is cloneable: each clone pulls from the same underlying channel. This gives us **multi-threaded execution for free**: spawn N threads, each running a clone of the executor, and tasks are distributed across them via the shared channel. That's the "MPMC" (multi-producer, multi-consumer) pattern.

⁵ `Send` means the future can be sent between threads; `'static` means it doesn't borrow from anything on the calling stack. These are fundamental requirements for multi-threaded executors, not limitations of our design. The spawned task outlives the stack frame that created it (hence `'static`), and may be polled on any executor thread (hence `Send`). For a deep dive into why relaxing these bounds is so hard, see the [Scoped Task Trilemma](https://without.boats/blog/the-scoped-task-trilemma/) discussed in Part 2.

⁶ `drop(sp)` drops the spawner's `Sender`. After the root task completes and the `Arc<Task>` is dropped (which releases the `loopback_entrance` sender), all senders are gone, the channel closes, and `ex.run()`'s `recv()` returns `Err`, ending the loop. Without this drop, the spawner's sender would keep the channel alive forever and the executor would block indefinitely.

⁷ This is the "async + parallelism" combination from [Part 1](/rust-async-demystified-p1#Concurrency-vs-Parallelism). Async gives us concurrency (interleaving tasks), and multiple executor threads give us parallelism (running tasks simultaneously on different cores). We got both almost for free, just by cloning the executor and handing clones to `std::thread::spawn`. The crossbeam channel handles all the synchronization.

⁸ A spurious wake happens when the reactor fires an event, but the I/O operation still returns `WouldBlock`. This can occur for several reasons: another thread might have consumed the data between the event and the retry, the OS might use edge-triggered notification where one event covers multiple state changes, or the OS might coalesce events. The retry loop handles all of these gracefully. In the worst case, we register the waker again and go back to sleep.

⁹ Both read and write share the same token, and thus the same waker slot. This means concurrent `read` + `write` on the same stream from different tasks isn't supported (the second `set_waker` call would overwrite the first). Production runtimes like Tokio solve this by maintaining separate waker slots for read and write readiness on each socket. For our echo server's sequential read-then-write pattern, this works perfectly.

¹⁰ Compare our `read()` implementation with the `TcpReadFuture` pseudocode in [Part 2, Piece 5](/rust-async-demystified-p2#The-Wake-up-Contract). The structure is identical: try `read()`, if `WouldBlock` then register the waker with the reactor and return `Pending`. The difference is that we use `async fn` + a loop instead of implementing `Future` by hand, and the compiler generates the state machine for us ([Part 2, Piece 3](/rust-async-demystified-p2#Piece-3-What-async-fn-Actually-Compiles-To)).

---

In *Rust Async Demystified* series:

- [Part 1 - Basic Concepts of Async Programming](/rust-async-demystified-p1)
- [Part 2 - Async Infrastructure in Rust and Other Languages](/rust-async-demystified-p2)
- Part 3 - Build Yourself a Minimal Runtime from Scratch (this article)

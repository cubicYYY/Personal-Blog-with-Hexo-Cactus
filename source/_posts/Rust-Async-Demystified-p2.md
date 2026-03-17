---
title: 'Rust Async Demystified: Part 2 - Async Infrastructure in Rust and Other Languages'
tags:
  - Rust
  - Asynchronous Programming
  - Tutorial
  - Rust Async Demystified
categories:
  - Programming-Languages
  - Rust
permalink: rust-async-demystified-p2/
date: 2026-03-08 20:00:00
---


> Author: CUBIC Y^3
> Feel free to share, but please credit the source and include a link to the original article. Thanks! :)

## Intro

In [Part 1](/rust-async-demystified-p1), we learned *what* asynchronous programming is and *why* we need it: to achieve concurrency without wasting time idling on I/O. We also saw the underlying mechanisms - callbacks, event-driven style - that make async possible.

But how does `async`/`await` in Rust actually work?

When you write `async fn` and `.await` in Rust, something almost magical happens. We'll peel back the layers of that "magic" one by one. Along the way, we'll compare Rust's design with Go and C++20 to understand the trade-offs each language makes.

By the end, you'll have a clear mental model of Rust's async infrastructure by completing a puzzle of following pieces:

- **Piece 1**: From callbacks to coroutines (state machine view)
- **Piece 2**: How Rust represents a coroutine — the `Future` trait
- **Piece 3**: What `async fn` actually compiles to
- **Piece 4**: Who calls `poll()` — the executor
- **Piece 5**: How futures get woken up — `Waker` and the reactor
- **Piece 6**: Why `Pin` exists — self-referential futures and safety

### Terminology (so we're on the same page)

Different languages use different words for the same idea. In this article we stick to this mapping:

| **Concept** | **What it means** | **In Rust** | **Elsewhere** |
| - | - | - | - |
| **Coroutine** | A function that can pause and resume, keeping its state. The general idea. | — | — |
| **The "thing" that represents one pausable computation** | The value/object you create when you start an async computation and that you later await or poll. | **`Future`** (the trait and the type implementing it) | JavaScript: **Promise**; C++20: **Task**; Go: **goroutine** |
| **Async** | The style of programming (non-blocking, concurrent). Adjective. | "async Rust", "async runtime" | Same idea everywhere |
| **`async fn` / `.await`** | Rust syntax for writing and consuming coroutines. | — | JS: `async`/`await`; C++20: `co_await`, etc. |

So: when we say **coroutine**, we mean the general concept. When we say **Future**, we mean Rust's concrete type for that concept. JavaScript's **Promise** is the analogous "thing you await"; C++20's **promise type** is something else (it's part of the coroutine machinery, not the same as a JS Promise). We'll use **Future** whenever we're talking about Rust's representation.

---

## Piece 1: From Callbacks to Coroutines

### The Problem We're Solving

In Part 1, we saw that callbacks lead to "callback hell" - nested, fragmented code that is hard to read and maintain. The dream is to write asynchronous code that *looks* like synchronous code. To achieve this, we need a way for a function to **pause** in the middle of its execution (when it would block on I/O) and **resume** later - without the programmer manually slicing logic into callback functions.

This is exactly what a **coroutine** is: a function that can suspend and resume in the middle of its body, preserving local state across suspensions. In Rust, each such pausable computation is represented by a **Future** (we'll see the type in Piece 2); you've already seen the syntax—`async fn` and `.await`.

Compare with a normal function:

| | **Normal function** | **Coroutine** |
| - | - | - |
| Execution | Call → return | Call → run → suspend → resume → … → return |
| Local state after return to the caller | Destroyed (stack frame popped) | Preserved |
| Resume in the middle | ❌ | ✅ |

Remember the coffee-shop analogy from [Part 1](/rust-async-demystified-p1)? To show how much cleaner coroutines are, suppose we now require customers to show the order ID to the barista to get the coffee.

With callbacks, you split the logic and must pass the `id` into the pickup callback explicitly:

```javascript
function orderCoffee(callback) {
    console.log("You order a coffee. The barista gives you a buzzer.");
    setTimeout(() => callback(/* order ID */ 42), 3000);
}

function visitCoffeeShop() {
    console.log("You enter the coffee shop.");
    orderCoffee((id) => {  // callback must accept id as parameter
        console.log("The buzzer buzzes! You pick up your coffee with order ID:", id, ". Enjoy!");
    });
    console.log("You are browsing a blog while waiting...");
}
```

A coroutine lets you write the whole flow in one function, and the `id` flows naturally into the continuation:

```rust
async fn order_coffee() -> u32 {
    println!("You order a coffee. The barista gives you a buzzer.");
    // Simulate waiting for the buzzer (order ready)
    async_std::task::sleep(std::time::Duration::from_secs(3)).await;
    42 // Order ID
}

async fn visit_coffee_shop() {
    println!("You enter the coffee shop.");

    let order_future = order_coffee();
    println!("You are browsing a blog while waiting...");  // runs immediately (cf. Part 1)

    let id = order_future.await;
    println!("The buzzer buzzes! You pick up your coffee with order ID: {id}. Enjoy!");
}
```

Unlike callbacks, you don't define and pass parameters into the callback function; it implicitly has access to all local variables (like `id`) in scope!

Under the hood, the compiler transforms this coroutine into a state machine that implements the `Future` trait (coming up next).

### A Coroutine Is Just a State Machine

How is "pausing in the middle of a function" even possible? The idea is simple: store the function's *progress* somewhere that survives across calls. Each time the function is called, check the progress and jump to the corresponding branch. That's a **state machine**. This state is kept until the coroutine finishes (no resumption after that). The compiler does this behind the scenes. When you write `async fn`, the compiler transforms it into a state machine: each `.await` becomes a state, and local variables that must survive across suspensions are saved in a struct.

Let's see a concrete example. This coroutine has two suspension points:

```rust
async fn fetch_and_save(url: &str, path: &str) {
    let data = http_get(url).await;   // suspension point 1
    file_write(path, &data).await;    // suspension point 2
    println!("Done!");
}
```

The compiler conceptually transforms it into a struct (holding all variables that live across any `.await`) plus a `poll()` method:

```rust
struct FetchAndSaveFuture {
    _state: u8,  // discriminant: which suspend point are we at?

    // All locals that need to survive across .await points live here.
    // Variables with non-overlapping lifetimes may share memory (like a union),
    // so the struct's size is the max across states, not the sum.
    url: MaybeUninit<String>,
    path: MaybeUninit<String>,
    http_future: MaybeUninit<HttpGetFuture>,
    data: MaybeUninit<Vec<u8>>,
    write_future: MaybeUninit<FileWriteFuture>,
}
```

You never see this struct directly - it's an anonymous, opaque type generated by the compiler. Only the fields relevant to the current state are valid; the rest are uninitialized. We use readable names here purely for explanation.

The `poll()` method drives this state machine forward:

```rust
impl Future for FetchAndSaveFuture {
    type Output = ();

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        loop {
            match self._state {
                0 => { // State `Start`: we have url and path
                    let url = unsafe { self.url.assume_init_read() };
                    self.http_future.write(http_get(&url));
                    self._state = 1;
                }
                1 => { // State `WaitingForHttp`: poll the HTTP sub-future
                    let fut = unsafe { self.http_future.assume_init_mut() };
                    match Pin::new_unchecked(fut).poll(cx) {
                        Poll::Pending => return Poll::Pending,
                        Poll::Ready(data) => {
                            let path = unsafe { self.path.assume_init_read() };
                            self.write_future.write(file_write(&path, &data));
                            self._state = 2;
                        }
                    }
                }
                2 => { // State `WaitingForWrite`: poll the file-write sub-future
                    let fut = unsafe { self.write_future.assume_init_mut() };
                    match Pin::new_unchecked(fut).poll(cx) {
                        Poll::Pending => return Poll::Pending,
                        Poll::Ready(()) => {
                            println!("Done!");
                            self._state = 3;
                            return Poll::Ready(());
                        }
                    }
                }
                _ => panic!("polled after completion"),
            }
        }
    }
}
```

> Don't worry about `Pin`, `Context`, or `Poll` yet - we'll cover them in the upcoming layers.

Each call to `poll()` tries to advance the state machine, starting from the current state (tracked by `_state`). If more work can be done (i.e., a sub-future is immediately ready), it continues progressing. If it reaches an operation that can't complete yet (like waiting on I/O), it returns `Poll::Pending`, effectively "pausing" at that point. When the future is woken (for example, when I/O completes), `poll()` is called again, and execution resumes from the paused state.

---

The most important takeaway: **`async` and `await` are just syntactic sugar**. The Rust compiler generates the struct + `poll()` automatically from your `async fn`. They don't introduce magic or fundamentally new concepts - rather, they encapsulate a well-established pattern for writing asynchronous code, in a more readable way.

While many languages use similar keywords like `await` and `async`, each language translates them into different underlying mechanisms and runtime behaviors. It's crucial to understand that `async`/`await` simply make async programming more ergonomic, but the actual implementation details vary significantly across languages.

### Async Runtime

You may have noticed something: the state machine doesn't run itself. The `poll()` function doesn't call itself - someone from the outside must keep calling it, checking whether it returned `Ready` or `Pending`, and deciding when to call it again. That "someone" is what we call an async **runtime** (or an "executor" in some runtimes). Without it, your `async fn` is just a data structure sitting in memory doing nothing.

This is a critical distinction: `async`/`await` defines the *what* (the state machine, i.e. the Future), but the runtime decides the *when* (when to poll, which task to run next, how to wait for I/O). Different languages make very different choices about this runtime, how they represent coroutines (Future, Promise, Task, etc.), and whether the programmer sees any of this at all. If you're curious how Rust, C++20, Go, and JavaScript compare side by side - with the same example implemented in all four - see [Appendix: Async Across Languages](#Appendix-Async-Across-Languages-Rust-C-JavaScript-and-Golang) at the end of this article.

Now, let's dive into Rust's machinery piece by piece.

---

## Piece 2: How Rust Represents a Coroutine — The `Future` Trait

We've been calling the "pausable function" a coroutine; in Rust that notion is represented by a type implementing the **`Future`** trait. So: coroutine (concept) → state machine (implementation) → **Future** (Rust's interface). The interface between that state machine and whatever drives it is the **`Future` trait**.

### The `Future` Trait Definition

Here's the actual definition from `std`:

```rust
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),
    Pending,
}
```

Let's walk through each piece:

- **`Output`** - the type of value this future eventually produces. An `async fn` that returns `String` becomes a `Future<Output = String>`.
- **`poll()`** - the single method that drives the state machine forward. Each call attempts to make progress.
  - Returns **`Poll::Ready(value)`** when the future has completed and the final value is available.
  - Returns **`Poll::Pending`** when the future cannot make progress right now (e.g., waiting for I/O). The runtime should *not* call `poll()` again until it's notified that progress is possible.
- **`Pin<&mut Self>`** - guarantees the future won't be moved in memory. We'll explain why in Piece 6.
- **`Context<'_>`** - carries a `Waker` that lets the future notify the runtime "I'm ready to be polled again." We'll cover this in Piece 5.

For now, you can think of `poll()` as asking the future: *"Can you make progress?"* The future answers either *"Yes, here's the result"* (`Ready`) or *"Not yet, I'll let you know"* (`Pending`).

### The Poll Model: Pull, Not Push

A `Future` does **nothing** until someone calls `poll()` on it. The future is **lazy**: creating it doesn't start any work. In some other languages, like JavaScript or Go, this is not the case.

There are two fundamental models for driving async tasks:

- **Push-based** (e.g. JavaScript Promises, Go goroutines): the task starts executing **immediately** when created and *pushes* results to you when done (via callbacks, channels, etc.). You don't control when work happens—it's already running.
- **Pull-based / poll-based** (Rust Futures): someone must actively *pull* by calling `poll()`. The Future makes progress **only** when polled. You create it, and nothing happens until you (or a runtime) start polling.

What are the trade-offs between these two models?

**Pull-based** (Rust's choice):

1. **Zero-cost cancellation**: to cancel a future, just stop polling it and drop it. No special cancellation protocol needed.
2. **No hidden allocations**: the future itself doesn't need to spawn background work or allocate a task slot.
3. **Composability**: combinators like `join!` and `select!` are straightforward - just poll sub-futures in the desired order.
4. **Zero-cost abstraction**: compiles down to a state machine with no runtime overhead.

**Push-based** (Go, JS):

1. **Immediate progress**: tasks start working as soon as they're created - no need to wire up an executor.
2. **No function coloring**: in Go, every function is the same "color" - the scheduler handles suspension transparently.
3. **Simpler mental model**: you don't think about polling, wakers, or executors.

> **Contrast with Go:** A goroutine starts running immediately when you `go func()`. It's push-based - the Go scheduler drives it. No explicit polling or awaiting required, but every goroutine allocates a stack (~2-8 KB initially) and is managed by the runtime's scheduler. Rust's poll model trades that implicit execution for explicit control and avoids the per-coroutine stack cost.

For more comparison, see [Appendix: Async Across Languages](#Appendix-Async-Across-Languages-Rust-C-JavaScript-and-Golang).

---

## Piece 3: What `async fn` Actually Compiles To

### `async` as Syntactic Sugar

When you write:

```rust
async fn fetch_data(url: &str) -> Data {
    let response = http_get(url).await;
    parse(response)
}
```

The compiler desugars this into an anonymous struct that implements `Future` - exactly the same transformation we saw in  [Piece 1](#Piece-1-From-Callbacks-to-Coroutines). The recipe is mechanical:

- Only local variables that **live across an `.await` point** are saved in the struct (variables used entirely within one state stay on the normal stack). The compiler may let variables with non-overlapping lifetimes share memory (like a union), so the struct's size can be the max across states rather than the sum.
- A `_state` discriminant tracking which `.await` we're at.
- A `poll()` method that does the real work: it polls child futures and transitions between states.

The `fetch_data` above has one `.await`, so it produces a struct with two states (start → waiting for HTTP) and fields for `url`, `http_future`, and `response`. The shape is identical to `FetchAndSaveFuture` from Layer 0, just with one fewer suspension point. We won't repeat the full desugared code here - scroll back to Layer 0 if you need a refresher.

The key takeaway: **`async fn` is just syntactic sugar**. Every `async fn` becomes an anonymous, opaque type that implements `Future`. You never see this type directly, but it's why you can pass futures around, store them in structs, and compose them - they're just values.

### Compare: Rust vs C++20 - Top-Down vs Bottom-Up

Both Rust and C++20 use stackless (we'll discuss this concept later) coroutines, which means they are all compiled into state machines. But they take opposite approaches to **how a parent coroutine drives its children**. This distinction shapes their respective APIs. (For a deeper treatment, see [this article](https://blog.howardlau.me/programming/coroutines-rust-cpp20.html).)

---

In Rust, the parent coroutine calls `poll()` on its child. If the child returns `Pending`, the parent also returns `Pending`. Next time the executor polls the parent, the parent re-enters the child by calling `poll()` again.

The parent always knows where its children are. No function pointers or handles are needed to "find your way back" - the call stack *is* the resumption path. The trade-off is that every poll traverses from the root coroutine all the way down to the leaf. But the API stays minimal: `Future` has a single method `poll()`, and composition is just nesting `poll()` calls.

```rust
// Rust: the child just returns Pending or Ready. That's it.
// It doesn't need to know anything about who called it.
impl Future for ChildFuture {
    type Output = Data;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Data> {
        // ... attempt work ...
        Poll::Pending // or Poll::Ready(data)
    }
}
```

---

In C++, when a child coroutine finishes or suspends, this child itself is responsible for telling the system what to call next. The child must store a pointer (or handle) to its parent so it can resume it. This is a bottom-up model.

Here's a simplified C example (adapted from [howardlau](https://blog.howardlau.me/programming/coroutines-rust-cpp20.html)) that shows the essence of this approach. Imagine `serve_http` needs to call a sub-coroutine `write_response`:

```c
// Each connection tracks which function to call and its state.
struct connection {
    void (*f)(void *state);  // current coroutine function
    void *state;             // current coroutine state
    int fd;
};

// The child coroutine stores a pointer back to its parent.
struct resp_state {
    int step;
    void (*parent)(void *);   // who to resume when I'm done
    void *parent_state;       // parent's state pointer
    // ... other fields ...
};

void write_response(struct resp_state *s) {
    switch (s->step) {
    case 0: /* write header ... */ s->step = 1; return;
    case 1: /* write body ...   */ break;
    }
    // Done! Resume the parent by rewiring the connection:
    // The event loop will call the parent next time.
    s->parent(s->parent_state);
}
```

The event loop doesn't know about parent-child relationships at all - it just calls whatever function pointer is registered:

```c
// Event loop: call the current coroutine for each I/O event.
for (/* each event p */) {
    struct connection *c = (struct connection *)p->data;
    c->f(c->state);  // could be serve_http OR write_response
}
```

C++20 formalizes this same pattern with `coroutine_handle`. But this is only *part* of the machinery. A C++20 coroutine also requires a **Promise type** (C++ jargon—unrelated to JavaScript's Promise; it's the object that controls the coroutine's lifecycle: `get_return_object()`, `initial_suspend()`, `final_suspend()`, `return_value()`/`return_void()`, `unhandled_exception()`) and a `coroutine_handle` to manage the coroutine's lifetime. The Awaitable below is just the part that controls `co_await` behavior:

```cpp
// C++20: the awaitable - just ONE piece of the coroutine machinery.
// You also need a Promise type, a return type wrapping coroutine_handle, etc.
struct ChildAwaitable {
    std::coroutine_handle<> parent_handle;

    bool await_ready() { return false; }
    std::coroutine_handle<> await_suspend(std::coroutine_handle<> parent) {
        parent_handle = parent;            // save who to resume
        return std::noop_coroutine();      // yield to the event loop
    }
    Data await_resume() { return data; }
};
```

(For a thorough walkthrough of the full C\+\+20 coroutine machinery, see [Yet Another C++ Coroutine Tutorial](https://joelschumacher.de/posts/yet-another-cpp-coroutine-tutorial).)

### Example: Nested Async Tasks in Rust and C++

![Rust vs C++ in Coroutine Design](Rust-TopDown-vs-Cpp-BottomUp.svg)

Rust:

1. **Executor Polls Main Future:** The async runtime (executor) starts the process by calling `poll()` on the top-level task (`Main Future`).
2. **Main Future Polls I/O Future:** The `Main Future` advances its state machine and hits an `.await` point, which invokes `poll()` on the nested `I/O Future`.
3. **Issue I/O:** The `I/O Future` initiates the actual asynchronous I/O operation with the operating system.
4. **Yield "Pending" (Child to Parent):** Because the I/O operation just started and the data isn't ready yet, the `I/O Future` returns a `Pending` state to the `Main Future`.
5. **Yield "Pending" (Parent to Executor):** The `Main Future` bubbles this `Pending` state back up to the executor, effectively putting the entire task chain to sleep and yielding control of the thread.
6. **Waker Signals Executor:** Once the background I/O operation completes, the I/O subsystem uses a `Waker` to call `Waker::wake()`. This acts as a notification to the executor that this specific task can make progress again.
7. **Executor Re-polls Main Future:** Reacting to the waker, the executor calls `poll()` on the `Main Future` from the top again.
8. **Main Future Re-polls I/O Future:** The `Main Future` resumes exactly where it left off, calling `poll()` on the `I/O Future` one more time.
9. **Return "Ready" with Data:** The `I/O Future` sees the completed data, retrieves it, and returns a `Ready` state along with the data to the `Main Future`.
10. **Return "Ready":** The `Main Future` finishes processing the data, completes its work, and returns a final `Ready` state to the executor, ending the task.

---

C++:

1. **Executor Resumes Main Task:** Execution begins when the executor (or caller) invokes `main_handle.resume()`, starting the top-level coroutine.
2. **Await Suspend & Handle Registration:** The `Main Task` evaluates an awaitable expression (`io()`). It checks if the result is ready via `io.await_ready()`. Since it returns `false`, the coroutine suspends itself and calls `io.await_suspend(main_handle)`. This critically passes its own continuation handle down, essentially saying, "Wake me up using this handle when you are done."
3. **Resume I/O Task:** Control transfers to the nested `I/O Task` via `io_handle.resume()`.
4. **Issue I/O:** The `I/O Task` kicks off the asynchronous I/O request to the system and then suspends itself. The calling thread is now completely free to go do other work.
5. **I/O Subsystem Resumes I/O Task:** When the I/O data is finally ready, the I/O subsystem (or an event loop) *directly* calls `io_handle.resume()`. The executor doesn't have to start from the top; execution jumps straight into the middle of the`I/O Task`.
6. **Return Value, Final Suspend, and Context Switch:**
    - **Inside the I/O Task:** The task processes the data and calls `return_value()` to store the result in its promise object. It then hits `final_suspend()`. Inside the `await_suspend` of this final step, it calls `main_handle.resume()`, directly handing execution control back to the `Main Task`.
    - **Inside the Main Task:** Now awake, the `Main Task` calls `io.await_resume()` to extract the result from the child's promise, and then `~io()` destroys the temporary awaiter object.
7. **Return to Executor:** The `Main Task` finishes processing the data and returns control back to the executor.

### The Trade-Off

| | **Top-down (Rust)** | **Bottom-up (C/C++20)** |
| - | - | - |
| Resumption path | Implicit (parent re-polls child) | Explicit (child stores pointer/handle to parent) |
| Child's knowledge of parent | None needed | Must store parent's handle |
| API surface | One method: `poll()` | C: manual function pointers; C++20: Promise type (~5 methods) + Awaitable (3 hooks) + `coroutine_handle` |
| Flexibility | Parent always drives | Child can resume *any* coroutine |
| Overhead | Re-enters from root to leaf each poll | Direct jump via handle (faster for deep chains) |
| Allocation | Future is a value (stack or `Box`) | Coroutine frame is heap-allocated by default |

Rust's top-down design keeps the API surface small, at the cost of re-traversing the state machine chain on every poll. The bottom-up design allows direct jumps to the leaf (potentially faster for deep chains) and gives the child full control over what gets resumed next, at the cost of a larger API surface and heap allocations.

---

## Piece 4: Who Calls `poll()`? - The Executor

A `Future` is inert - it needs someone to call `poll()`. That "someone" is the **executor** (also called the async runtime).

### What an Executor Does

At its simplest, an executor is a loop that drives your futures to completion. Here's what it does:

1. **Maintain a task queue** of top-level futures (each future you `spawn()` becomes a *task*).
2. **Poll each ready task** by calling `task.poll(cx)`.
3. **If a task returns `Pending`**, the executor parks it - it won't be polled again until something wakes it up.
4. **When a task is woken** (via the `Waker` mechanism - more in Piece 5), the executor moves it back into the ready queue.
5. **Repeat** until all tasks complete.

Here's a drastically simplified executor loop to build intuition (a real one has ~200 more lines of machinery):

```rust
// Pseudocode - not real Rust
fn run(tasks: Vec<Task>) {
    let ready_queue = Queue::from(tasks);

    loop {
        while let Some(task) = ready_queue.pop() {
            let waker = create_waker_for(&task, &ready_queue);
            let cx = &mut Context::from_waker(&waker);

            match task.future.poll(cx) {
                Poll::Ready(result) => { /* task is done, drop it */ }
                Poll::Pending        => { /* parked - waker will re-enqueue it */ }
            }
        }
        // Nothing ready? Sleep until the OS signals an I/O event (via the reactor).
        wait_for_io_events();
    }
}
```

This executor doesn't busy-loop over all futures. It only polls futures that have been *woken up* - meaning something told the executor "this one can make progress now." This is what makes the poll model efficient.

### Why Rust Has No Built-in Runtime

Rust is a *systems language* - it targets everything from embedded microcontrollers to OS kernels to high-throughput web servers. A one-size-fits-all runtime would inevitably be wrong for many of these use cases.

Instead, Rust defines the *interface* (`Future`, `Waker`, `Poll`) and lets **you** choose a runtime that fits your constraints:

| Runtime | Description |
| - | - |
| **[tokio](https://tokio.rs/)** | Multi-threaded, work-stealing scheduler. The most popular choice for server applications. |
| **[async-std](https://async.rs/)** | Similar API to `std`, multi-threaded. |
| **[smol](https://github.com/smol-rs/smol)** | Lightweight, minimal dependencies. |
| **[embassy](https://embassy.dev/)** | Designed for embedded / `no_std` environments - no heap allocator required. |

This is Rust's "bring your own runtime" philosophy. The same `Future` trait works on a Cortex-M microcontroller with `embassy` and a 128-core server with `tokio`. The trade-off: you must pick (and depend on) a runtime, and different runtimes have subtly different behaviors.

For example, `tokio::spawn` requires `'static` futures - you can't borrow from the parent task's stack. This isn't a tokio limitation; it's a fundamental constraint. without.boats calls it the [Scoped Task Trilemma](https://without.boats/blog/the-scoped-task-trilemma/): any sound async API can provide at most two of *concurrency*, *parallelizability*, and *borrowing*. `tokio::spawn` provides concurrency + parallelizability, so it must sacrifice borrowing - hence the `'static` bound.

> **Contrast with Go and JS:**
>
> - Go has a **built-in M:N scheduler** that maps goroutines onto OS threads with work-stealing. No executor choice needed - `go func()` runs on the built-in scheduler.
> - JavaScript has a **built-in event loop** baked into the browser/Node.js runtime.
> - Rust requires you to choose (and depend on) a runtime. In return, the same `Future` trait works across vastly different platforms - from a microcontroller with `embassy` to a 128-core server with `tokio`.

---

## Piece 5: How Does a Future Get Woken Up? - The `Waker`

When `poll()` returns `Pending`, the executor needs to know *when* to poll again. It would be wasteful to poll every future in a loop (that's just busy-waiting!). This is where the **`Waker`** comes in.

### The Wake-up Contract

Remember the `Context<'_>` parameter in `poll()`? It carries a **`Waker`** - a handle that the future can use to say "wake me up later." Here's how the contract works:

1. The executor creates a `Waker` for each task and passes it into `poll()` via `Context`.
2. If the future can't make progress (e.g., the socket has no data yet), it **clones and stores the `Waker`** somewhere - typically by registering it with the I/O subsystem.
3. Later, when the underlying event occurs (data arrives on a socket, a timer expires, etc.), the `Waker::wake()` method is called.
4. This notifies the executor: *"Hey, this task can make progress now - put it back in the ready queue."*
5. The executor re-polls the future, which can now advance to its next state.

The `Waker` is the *only* way a `Pending` future gets polled again. No waker call → no re-poll → the future sits idle forever. This is why the pull model is efficient: the executor never wastes time polling futures that aren't ready.

Here's what a leaf future (the bottom of the chain - the one that actually talks to the OS) looks like:

```rust
impl Future for TcpReadFuture {
    type Output = Vec<u8>;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Vec<u8>> {
        match self.socket.try_read(&mut self.buf) {
            Ok(n)  => Poll::Ready(self.buf[..n].to_vec()),
            Err(e) if e.kind() == WouldBlock => {
                // No data yet. Register the waker so the reactor
                // will wake us when data arrives on this socket.
                self.reactor.register(self.socket.fd, cx.waker().clone());
                Poll::Pending
            }
            Err(e) => panic!("read error: {e}"),
        }
    }
}
```

### The Reactor: Connecting to the OS

The **reactor** is the component that interfaces with OS-level I/O notification systems (`epoll` on Linux, `kqueue` on macOS, `IOCP` on Windows, `io_uring` on modern Linux).

These are essentially **OS-level callback mechanisms**: you tell the kernel "notify me when something happens on this file descriptor," and the kernel does exactly that - without your program having to spin in a loop checking. This is where the real efficiency of async comes from. Without these facilities, the only option would be busy-waiting (repeatedly checking "is it ready yet?"), which wastes CPU cycles. In the earliest days of computing, programs had no choice but to wait for each I/O operation to complete before moving on. OS-level event notification is what makes modern cooperative multitasking practical.

Its job:

1. Register interest in I/O events (e.g., "wake me when socket #42 has data").
2. Block efficiently at the OS level, waiting for any registered event to fire.
3. When events arrive, look up the corresponding `Waker` and call `waker.wake()` to notify the executor.

> We won't dive into OS notification APIs like `epoll` or `io_uring` in this article. The key takeaway: **they are the bridge that carries asynchrony from the hardware level up into your program**.

The reactor is usually bundled inside the runtime (e.g., tokio includes both an executor and a reactor), but conceptually they are separate components with a clean division of labor.

### Putting Executor + Waker + Reactor Together

Here's the full cycle for a single I/O operation, assuming that `epoll` is used by OS:

![Executor, Reactor and Future](executor-reactor-future.svg)

1. **Executor Polls the Future:** The process begins when the Executor (the scheduler) calls `poll(cx)` on the Future. It passes along a context (`cx`), which contains the `Waker` needed to wake the task up later.
2. **Future Initiates I/O:** The Future attempts a non-blocking I/O system call directly to the OS. Because the data isn't instantly available, the OS returns an error like `EWOULDBLOCK` or `EAGAIN`.
3. **Future Registers with the Reactor:** Realizing it must wait, the Future registers its file descriptor (`fd`) and the `Waker` from its context with the Reactor. It is essentially saying, "I can't make progress right now. Please monitor this `fd` and use this `waker` to notify the Executor when the OS says the data is ready."
   - Behind the scenes, the Reactor uses `epoll_ctl` to instruct the OS to watch the `fd`, and loops `epoll_wait` to listen for any `fd` registered is ready.
4. **Future Yields Control:** The Future returns a `Pending` state to the Executor. The Executor safely puts this task to sleep and frees up the CPU thread to work on other concurrent tasks.
5. **Reactor Wakes the Task:** Once the OS finishes the background I/O work (e.g., data arrives over the network), it notifies the Reactor. The Reactor looks up the `Waker` associated with that specific `fd` and calls `waker.wake()`. This places the suspended task back onto the Executor's ready queue.
6. **Executor Re-polls the Future:** Seeing the task in its ready queue, the Executor resumes execution by calling `poll(cx)` on the Future a second time.
7. **Future Retrieves Data:** The Future reaches out to the OS once more to perform the I/O read/write operation. Because the OS has already buffered the data, the operation succeeds immediately, yielding the final I/O result.
8. **Future Completes:** The Future returns `Ready` (along with the resulting data) back to the Executor. The task is now officially complete.

This three-way collaboration - **executor** (schedules tasks), **future** (state machine), **reactor** (bridges OS I/O) - is the heart of Rust's async runtime. The `Waker` is the glue: it's how the reactor tells the executor "this future is ready to be polled again," without the executor ever busy-waiting — it is essentially a callback function.

In other words, callbacks haven't disappeared at all; they're just pushed down a layer. Instead of you writing `on_readable(fd, callback)` manually, the reactor and OS keep an internal table of "when this fd is ready, call this waker," and the executor treats "task is ready" as its own callback-like signal. `async`/`await` mostly hides these callbacks behind the `Future`/`Waker` API and the executor's event loop so that your application code can stay in a direct, sequential style while callbacks quietly drive progress underneath.

> **Contrast with Go:** Go's runtime handles all of this internally. When a goroutine calls a blocking I/O function, the Go runtime transparently parks the goroutine and integrates with `epoll`/`kqueue`/`IOCP` under the hood. You never see `Waker` or `Reactor` - it's all invisible. In Rust, these layers are explicit and separable - you can swap reactors, customize waker behavior, or write your own. The trade-off is that you need to understand these components to use async effectively.

---

## Piece 6: Why `Pin`? - The Self-Referential Problem

You may have noticed `Pin<&mut Self>` in the `Future::poll` signature and wondered what it's about. This is one of Rust's most confusing async concepts, but it exists for a very good reason.

### The Problem

Recall that the compiler turns an `async fn` into a struct whose fields hold every local variable that lives across an `.await`. Now consider this code:

```rust
async fn process() {
    let data = vec![1, 2, 3];
    let slice = &data[..];     // `slice` borrows from `data`
    send_to_network(slice).await;  // both `data` and `slice` must survive this .await
}
```

After desugaring, the state machine struct looks roughly like:

```rust
struct ProcessFuture {
    _state: u8,
    data: Vec<u8>,
    slice: *const [u8],  // points into `data` above!
    // ...
}
```

The field `slice` is a pointer *into* `data` - both are fields of the *same struct*. This is a **self-referential struct**.

Now here's the danger: if someone moves this struct (e.g., by returning it from a function, pushing it into a `Vec`, or calling `std::mem::swap`), `data` gets a new address in memory - but `slice` still points at the *old* address. **Dangling pointer. Undefined behavior.**

In normal Rust, moving values is always safe because Rust doesn't have self-referential borrows. But compiler-generated futures *do* have them. We need a way to tell the compiler: **"this value must not be moved once it has been set up."**

That's exactly what `Pin` does.

### How `Pin` Works (Briefly)

`Pin<&mut T>` is a wrapper around a mutable reference that **prevents you from getting a plain `&mut T` back** (which would allow moving via `std::mem::swap` or `std::mem::replace`). The key rules:

- Once a value is pinned, you can only access it through `Pin<&mut T>`, which doesn't let you move the underlying value.
- Types that are **safe to move** (most types) can implement the `Unpin` marker trait, which opts out of pinning restrictions. For `Unpin` types, `Pin<&mut T>` behaves just like `&mut T`.
- Most leaf futures (like I/O primitives, timers) are `Unpin`. It's the **compiler-generated state machines** - the ones that may contain self-references - that are `!Unpin` and genuinely need pinning.

This is why `poll()` takes `self: Pin<&mut Self>`: the executor must pin the future before polling it, guaranteeing the state machine won't move while self-references exist. In practice, you rarely interact with `Pin` directly - `Box::pin()`, `tokio::pin!()`, or the runtime handle it for you.

> **Contrast with Go:** Goroutines use heap-allocated, GC-managed stacks, so pointer validity is handled by the garbage collector. No pinning concept is needed, but this requires GC overhead.

---

## Zooming Out: Stackful vs Stackless and the Design Trade-off

Now that we've seen Rust's layered design in detail, let's zoom out and look at the fundamental design choice that underpins everything above.

### Two Families of Coroutines

- **Stackless coroutines** (Rust, C++20, JavaScript): The coroutine has **no separate stack**. Instead, the compiler generates a state machine struct that holds only the variables needed across suspension points. The coroutine can only suspend at explicit `await` points. Memory footprint: just the size of the state machine struct (often tens to hundreds of bytes).

- **Stackful coroutines** (Go goroutines, Lua coroutines, Java virtual threads): Each coroutine has **its own stack** (a separate block of memory, typically 2-8 KB initially, growable). The coroutine can suspend from *anywhere* in the call stack - even deep inside nested function calls that know nothing about async. This means no function coloring, but each coroutine carries a per-stack memory cost.

| | **Stackless** | **Stackful** |
| - | - | - |
| Memory per coroutine | Size of the state machine (~bytes to ~KB) | A full stack (~2-8 KB minimum) |
| Suspend from | Only explicit `await` points | Anywhere, including nested calls |
| Compiler involvement | Heavy (generates state machine) | Minimal (runtime does the work) |
| Examples | Rust, C++20, JavaScript | Go, Lua, Java virtual threads |

### Why Rust Chose Stackless

There's a pattern: languages with a garbage collector (Go, Java) tend to adopt stackful coroutines, since the GC already manages heap-allocated stacks. Languages without a GC (Rust, C++20) tend to adopt stackless coroutines to avoid that runtime cost.

Rust is a systems language that prizes **zero-cost abstractions** - you shouldn't pay for what you don't use. Stackless coroutines fit this principle:

1. **No hidden allocations**: the state machine struct can live on the stack, in a `Box`, or in a static buffer. You choose.
2. **Predictable performance**: you know exactly what each `.await` compiles to - a state transition in a `match` statement. No context switches, no stack copying.
3. **Embeddable**: works on `no_std`, bare-metal, embedded microcontrollers - no need for a stack allocator or virtual memory.
4. **Composability**: since futures are just structs implementing a trait, you can nest, combine, and transform them with zero overhead.

The cost is explicitness: you must mark every async boundary with `async fn` and `.await`. This is known as the "function coloring" problem.

### The Function Coloring Problem

In Rust (and JavaScript, and C++20), `async fn` and regular `fn` are fundamentally **different "colors"** - you can't transparently call one from the other:

- An `async fn` can call a regular `fn` ✅ (no issue - synchronous code just runs)
- A regular `fn` **cannot** `.await` an `async fn` ❌ (you need an executor to poll it)

This means once part of your call stack becomes async, the async-ness tends to **propagate upward**: the caller must also be `async`, and its caller, and so on - all the way up to the top-level executor entry point (e.g., `#[tokio::main]`).

```rust
async fn inner() -> i32 { 42 }

// ❌ This won't compile:
fn outer() -> i32 {
    inner().await  // error: `.await` is only allowed inside `async fn`
}
```

Go avoids coloring entirely: because the runtime handles suspension invisibly at any point, every function is "the same color." You never think about whether a function is async or not - the Go scheduler handles it. The trade-off is the runtime overhead (stack allocation, scheduler bookkeeping) discussed above.

Rust accepts coloring as the **price for zero-cost abstraction and explicitness**: every `.await` is a visible suspension point, so you always know where your code might pause. This matters when reasoning about shared state, locks, and cancellation - but it does propagate through your codebase. (For more on this trade-off, see Bob Nystrom's essay [*What Color is Your Function?*](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/))

---

## Practical Implications of Rust's Design

Understanding the layers above gives you practical wisdom for writing async Rust.

### Laziness: Nothing Happens Until You `.await`

In Rust, creating a future does **not** start any work. The future is an inert state machine sitting in memory until someone polls it. This catches many newcomers off guard:

```rust
async fn do_work() {
    println!("working...");
}

async fn main_task() {
    do_work();  // ⚠️ WARNING: this does nothing! The future is created and immediately dropped.
}
```

The Rust compiler even warns you: *"unused future that must be used - futures do nothing unless you `.await` or poll them."*

If you want to run a future concurrently (without awaiting it inline), you must **spawn** it as a task:

```rust
// Run two tasks concurrently:
let handle = tokio::spawn(do_work());  // starts the task on the executor
// ... do other things ...
handle.await;  // wait for it to finish
```

> **Contrast**: In JavaScript, `fetchData()` starts executing immediately and returns a `Promise` that's already in-flight. In Go, `go doWork()` launches a goroutine that runs immediately. In Rust, you must be explicit about when execution begins.

### Lifetime Challenges Across `await` Points

Since the compiler captures local variables into a state machine struct, any reference held across an `.await` must remain valid for the entire duration of the future. This leads to situations that feel surprising if you're not thinking in terms of state machines:

```rust
async fn problematic() {
    let data = vec![1, 2, 3];
    let reference = &data[0];  // borrows from `data`
    
    some_async_op().await;     // `reference` must survive across this .await
    
    println!("{}", reference); // still using the borrow
}
```

The compiler may reject this if it can't prove the borrow is safe across the suspension point. In practice, a common workaround is to **clone** the value or restructure the code so the borrow doesn't span an `.await`:

```rust
async fn fixed() {
    let data = vec![1, 2, 3];
    let value = data[0];       // copy the value, don't borrow
    
    some_async_op().await;
    
    println!("{}", value);     // no borrow to worry about
}
```

A particularly common trap is holding a `MutexGuard` across an `.await` - see [Async-Aware Synchronization](#Async-Aware-Synchronization) below.

> **Contrast:** Go's garbage collector ensures references remain valid as long as they're reachable - no lifetime issues. C++ has the same self-reference problem as Rust, but without compile-time enforcement - dangling references are possible and are not caught until runtime (if at all).

### Cancellation via `Drop`

Since a future is just a value (a struct), **dropping it cancels it**. The state machine is destroyed, its fields are dropped, and any resources it held are freed. No special cancellation protocol needed.

This powers patterns like `tokio::select!`, which polls multiple futures concurrently and drops the "losers":

```rust
tokio::select! {
    result = fetch_from_server_a() => println!("A responded: {:?}", result),
    result = fetch_from_server_b() => println!("B responded: {:?}", result),
    // Whichever finishes second is automatically dropped (cancelled).
}
```

**Caveat**: Rust does not (yet) have `async Drop`. If your future needs to perform async cleanup when cancelled (e.g., sending a "goodbye" message over a network), you can't do it in `Drop` (which is synchronous). You need to handle this with explicit cleanup logic or wrapper types.

> **Contrast with Go:** Cancellation in Go uses `context.Context`, threaded through every function in the chain. Each function must explicitly check `ctx.Done()` to cooperate. Forgetting to check means the goroutine keeps running. Rust's drop-based cancellation is automatic but limited to synchronous cleanup (no `async Drop` yet); Go's `Context` propagation is manual but supports arbitrary cleanup.
>
> **Contrast with C++:** Destroying a `coroutine_handle` destroys the coroutine frame. However, the programmer must ensure no other handle still references it - there's no compiler-enforced ownership model like Rust's `Drop`.

### Async-Aware Synchronization

A common trap: using `std::sync::Mutex` in async code. The lock itself works fine, but holding the `MutexGuard` **across an `.await`** is dangerous:

```rust
let guard = mutex.lock().unwrap();
some_async_op().await;  // ⚠️ The executor thread is blocked while holding the lock!
drop(guard);
```

While this task is suspended at the `.await`, the `MutexGuard` is still held - and since `std::sync::Mutex` is a blocking lock, any other task on the same executor thread that tries to acquire it will **deadlock** (especially on a single-threaded runtime).

The solution is to use **async-aware locks** like `tokio::sync::Mutex`, which yield to the executor when the lock is contended:

```rust
let guard = mutex.lock().await;  // async lock: yields instead of blocking
some_async_op().await;           // safe - other tasks can still run
drop(guard);
```

**Rule of thumb**:

- Use `std::sync::Mutex` for short, synchronous critical sections (lock → do quick work → unlock, no `.await` inside).
- Use `tokio::sync::Mutex` (or equivalent) when you must hold a lock across `.await` points.

This illustrates a broader point: in async Rust, you generally need to use **async-aware versions** of blocking operations (sleep, I/O, locks) provided by your runtime, because the executor and reactor need to cooperate. Using blocking OS primitives directly will stall the executor thread.

> **Contrast with Go:** Go uses channels as the primary synchronization primitive ("share memory by communicating"), and the runtime transparently migrates goroutines across OS threads. Holding a `sync.Mutex` across a blocking call doesn't stall other goroutines, because Go's scheduler can move them to a different OS thread. In Rust, you must be aware of this distinction yourself.

---

## Summary

- **Coroutines as state machines**: An `async fn` compiles to a state machine struct that preserves local state across `.await` points and is driven by a single `poll()` method.
- **Futures are lazy and poll-based**: Creating a future does nothing until an executor polls it; Rust chooses a pull model, unlike Go goroutines or JavaScript Promises.
- **Executor, reactor, waker**: The executor schedules tasks and calls `poll()`, the reactor integrates with OS I/O notifications, and `Waker` connects them so futures are only polled when work can actually proceed.
- **Stackless design and `Pin`**: Rust uses stackless coroutines (no per-coroutine stack) and `Pin` to keep self-referential futures from moving, trading runtime overhead for compile-time guarantees.
- **Function coloring and ergonomics**: `async fn` and regular `fn` are different "colors," which can propagate through your call stack, but in return you get explicit suspension points and predictable performance.

## What's Next?

We've demystified the *design* - now it's time to see the *code*. In Part 3, we'll **build a minimal async runtime in Rust from scratch**: a simple executor, a reactor backed by OS I/O notifications, and a `Waker` implementation. You'll see every layer we discussed come to life in ~200 lines of Rust.

---

## References

[Coroutines: Rust and C++20 - CodeTalks (howardlau)](https://blog.howardlau.me/programming/coroutines-rust-cpp20.html)
[What Color is Your Function? - Bob Nystrom](https://journal.stuffwithstuff.com/2015/02/01/what-color-is-your-function/)
[Asynchronous Programming in Rust](https://rust-lang.github.io/async-book/)
[The Rust `Future` trait](https://doc.rust-lang.org/std/future/trait.Future.html)
[C++20 Coroutines - cppreference](https://en.cppreference.com/w/cpp/language/coroutines)
[Yet Another C++ Coroutine Tutorial - Joel Schumacher](https://joelschumacher.de/posts/yet-another-cpp-coroutine-tutorial)
[Concurrency in Go - Effective Go](https://go.dev/doc/effective_go#concurrency)
[The Scoped Task Trilemma - without.boats](https://without.boats/blog/the-scoped-task-trilemma/)
[Tokio Tutorial](https://tokio.rs/tokio/tutorial)

Figures are created using [Excalidraw](https://excalidraw.com/).

---

## Appendix: Async Across Languages: Rust, C++, JavaScript and Golang

All four languages below let you write code that "pauses" and "resumes" - but the machinery underneath varies dramatically.

| | **Rust** | **C++20** | **Go** | **JavaScript** |
| - | - | - | - | - |
| Coroutine type | Stackless | Stackless | Stackful (goroutines) | Stackless |
| Core abstraction | `Future` trait | C++20 coroutine (Promise type + handle) | Goroutine + Channel | `Promise` |
| Keywords | `async fn`, `.await` | `co_await`, `co_yield`, `co_return` | (none - implicit) | `async`, `await` |
| Driving model | Top-down (parent polls child) | Bottom-up (child resumes parent via handle) | Built-in Go scheduler | Built-in event loop |
| Who runs it? | You choose a runtime (tokio, smol, ...) | You choose a library (Asio, libunifex, ...) | Built-in Go scheduler | Built-in event loop |
| Task starts when? | When first polled (lazy) | When first `co_await`'d (lazy) | Immediately on `go f()` (eager) | Immediately on creation (eager) |
| Cancellation | Drop the future | Destroy the coroutine handle | `context.Context` | `AbortController` |
| Function coloring | Yes (`async fn` ≠ `fn`) | Yes (coroutine ≠ function) | No (all functions are the same) | Yes (`async` ≠ regular) |
| Memory safety | Compile-time (`Pin` + ownership) | Manual (UB possible) | GC | GC |

Every row represents a trade-off, not a winner. Rust consistently chooses **explicitness and zero-cost abstractions**, accepting a steeper learning curve in exchange for fine-grained control. Go consistently chooses **implicit runtime management**, accepting per-goroutine overhead in exchange for a uniform programming model. C++ exposes **maximum flexibility** in the coroutine protocol, accepting API complexity. JavaScript provides a **built-in event loop**, accepting the single-threaded constraint. The right choice depends on the constraints of your project.

### Rust: `async fn` → compiler-generated state machine (pulled by a runtime)

```rust
// What you write:
async fn fetch_and_save(url: &str, path: &str) {
    let data = http_get(url).await;
    file_write(path, &data).await;
}

// To actually run it, you need a runtime:
#[tokio::main]
async fn main() {
    fetch_and_save("https://example.com", "out.txt").await;
}
```

De-sugared: the compiler produces a flat struct with a discriminant + `poll()`. The tokio runtime calls `poll()` repeatedly (top-down), parking the task on `Pending` and waking it when I/O is ready.

```rust
// What the compiler generates (conceptual):
struct FetchAndSaveFuture {
    _state: u8,
    url: MaybeUninit<String>,
    path: MaybeUninit<String>,
    http_future: MaybeUninit<HttpGetFuture>,
    data: MaybeUninit<Vec<u8>>,
    write_future: MaybeUninit<FileWriteFuture>,
}

impl Future for FetchAndSaveFuture {
    type Output = ();
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        loop {
            match self._state {
                0 => {
                    self.http_future.write(http_get(&self.url));
                    self._state = 1;
                }
                1 => match Pin::new_unchecked(&mut self.http_future).poll(cx) {
                    Poll::Pending => return Poll::Pending,
                    Poll::Ready(data) => {
                        self.write_future.write(file_write(&self.path, &data));
                        self._state = 2;
                    }
                },
                2 => match Pin::new_unchecked(&mut self.write_future).poll(cx) {
                    Poll::Pending => return Poll::Pending,
                    Poll::Ready(()) => return Poll::Ready(()),
                },
                _ => unreachable!(),
            }
        }
    }
}
```

### C++20: `co_await` → compiler-generated coroutine frame (pulled by a library)

```cpp
// What you write:
Task<void> fetch_and_save(std::string url, std::string path) {
    auto data = co_await http_get(url);
    co_await file_write(path, data);
}

// You need a library (e.g., Asio) to drive the coroutine.
int main() {
    asio::io_context ctx;
    asio::co_spawn(ctx, fetch_and_save("https://example.com", "out.txt"), asio::detached);
    ctx.run(); // the event loop that calls resume() on suspended coroutines
}
```

De-sugared: the compiler allocates a **coroutine frame** on the heap (by default). Each `co_await` checks `await_ready()` → `await_suspend()` → `await_resume()` on the awaitable. The child stores the parent's `coroutine_handle` and resumes it when done (bottom-up). Like Rust, no built-in runtime - you must provide one.

```cpp
// What the compiler generates (conceptual, all pieces shown):

// ── Piece 1: The return type + Promise type ──────────────────────
// You must define these. The compiler looks for promise_type inside Task<T>.

struct Task {
    struct promise_type {
        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; } // lazy start
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> handle;

    void resume() { handle.resume(); }
    void destroy() { handle.destroy(); }
};

// ── Piece 2: The Awaitable ───────────────────────────────────────
// Each co_await expression needs an awaitable with these 3 methods.
// This one wraps http_get: it suspends, does work, then resumes the caller.

struct HttpGetAwaitable {
    std::string url;
    std::vector<uint8_t> result;
    std::coroutine_handle<> parent;  // stores who to resume (bottom-up!)

    bool await_ready() { return false; }  // not ready yet, must suspend
    void await_suspend(std::coroutine_handle<> h) {
        parent = h;                       // save the parent's handle
        // ... schedule the actual HTTP work; when done, call parent.resume()
    }
    std::vector<uint8_t> await_resume() { return std::move(result); }
};

// ── Piece 3: The coroutine frame (compiler-generated) ────────────
// The compiler heap-allocates a frame holding all state:

struct fetch_and_save_frame {
    void (*resume_fn)(fetch_and_save_frame*);   // pointer to resume logic
    void (*destroy_fn)(fetch_and_save_frame*);  // pointer to cleanup logic
    Task::promise_type promise;
    int suspend_point;        // which co_await are we at? (0, 1, 2, ...)

    // Captured locals:
    std::string url, path;
    std::vector<uint8_t> data;

    // Sub-awaitables (one per co_await):
    HttpGetAwaitable http_awaitable;
    FileWriteAwaitable write_awaitable;
};

// ── Piece 4: The resume function (compiler-generated) ────────────
// Each call to handle.resume() jumps into this switch:

void fetch_and_save_resume(fetch_and_save_frame* f) {
    switch (f->suspend_point) {
    case 0: // initial suspend (from initial_suspend())
        f->http_awaitable = HttpGetAwaitable{f->url};
        if (!f->http_awaitable.await_ready()) {
            f->suspend_point = 1;
            f->http_awaitable.await_suspend(/* my handle */);
            return;  // suspended - control returns to the event loop
        }
        [[fallthrough]];
    case 1: // resumed after http_get
        f->data = f->http_awaitable.await_resume();
        f->write_awaitable = FileWriteAwaitable{f->path, f->data};
        if (!f->write_awaitable.await_ready()) {
            f->suspend_point = 2;
            f->write_awaitable.await_suspend(/* my handle */);
            return;  // suspended
        }
        [[fallthrough]];
    case 2: // resumed after file_write
        f->write_awaitable.await_resume();
        f->promise.return_void();
        f->suspend_point = 3; // final suspend
        return;
    }
}
```

### JavaScript: `async`/`await` → Promise chain (pushed by the built-in event loop)

In JS, the analogue of Rust's Future is a **Promise**—the value you `await` and that runs on the event loop.

```javascript
// What you write:
async function fetchAndSave(url, path) {
    const data = await httpGet(url);
    await fileWrite(path, data);
}

// Just call it - the built-in event loop handles everything:
fetchAndSave("https://example.com", "out.txt");
```

De-sugared: each `await` splits the function into continuation callbacks chained via `Promise.then()`. The built-in event loop (in the browser or Node.js) drives execution. Note: the Promise starts executing **immediately** when created.

```javascript
// What the engine conceptually transforms this into:
function fetchAndSave(url, path) {
    return httpGet(url)               // returns a Promise (starts immediately)
        .then(function (data) {       // continuation 1: called when httpGet resolves
            return fileWrite(path, data);
        })
        .then(function () {           // continuation 2: called when fileWrite resolves
            // done
        });
}

// The event loop picks up resolved Promises from the microtask queue
// and calls the next .then() callback. No polling, no state machine -
// just a chain of callbacks managed by the runtime.
```

### Go: implicit goroutine (pushed by the built-in scheduler)

```go
// What you write - no special keywords at all:
func fetchAndSave(url string, path string) {
    data := httpGet(url)   // blocks the goroutine, not the OS thread
    fileWrite(path, data)  // same
}

// Launch it as a goroutine:
func main() {
    go fetchAndSave("https://example.com", "out.txt")
    // ... other work ...
    select {} // keep main alive
}
```

De-sugared: **there is no de-sugaring**. Go doesn't transform your function at all. Instead, each goroutine gets its own stack (~2-8 KB, growable). When `httpGet` blocks on I/O, the Go runtime's scheduler parks the goroutine and switches to another one on the same OS thread. No state machine, no explicit polling - the runtime does everything behind the scenes.

```text
// There is no transformed code to show.
// The Go scheduler does this at runtime:

goroutine G1 calls httpGet(url)
    → httpGet issues non-blocking I/O syscall
    → I/O not ready yet
    → Go runtime parks G1 (saves its stack pointer & program counter)
    → Go runtime picks another runnable goroutine G2 from the run queue
    → G2 resumes on the same OS thread

... later, the OS signals that G1's socket has data ...

    → Go runtime's netpoller (backed by epoll/kqueue) notices
    → G1 is moved back to the run queue
    → Eventually G1 is scheduled onto an OS thread and resumes
    → httpGet returns the data to G1 as if it had blocked normally
```

---

## Further Reading

If you'd like to go deeper into some of the design trade-offs we touched on, these are excellent reads:

- [**The Scoped Task Trilemma** - without.boats](https://without.boats/blog/the-scoped-task-trilemma/): Why any sound async concurrency API in Rust can provide at most two of *concurrency*, *parallelizability*, and *borrowing*. Explains why `tokio::spawn` requires `'static`, why `select!` can't parallelize, and why `thread::scope` blocks the parent.
- [**Pre-Pooping Your Pants With Rust** - Alexis Beingessner](https://cglab.ca/~abeinges/blah/everyone-poops/): The story of the 2015 "Leakpocalypse" - how `Rc` + `RefCell` can leak destructors in safe code, why `mem::forget` became safe, and the practical patterns for writing destructor-based APIs that stay sound even when destructors don't run. Directly relevant to understanding why Rust's drop-based cancellation (Layer 5) works the way it does.

---

In *Rust Async Demystified* series:

- [Part 1 - Basic Concepts of Async Programming](/rust-async-demystified-p1)
- Part 2 - Async Infrastructure in Rust and Other Languages (this article)
- [Part 3 - Build Yourself a Minimal Runtime from Scratch](/rust-async-demystified-p3)

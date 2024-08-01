---
title: 'Rust Async Demystified: Part 1 - Basic Concepts of Asynchronous Programming'
tags:
  - Rust
  - Programming
  - Tutorial
  - Async
categories:
  - Programming-Languages
  - Rust
permalink: rust-async-demystified-p1/
date: 2024-08-02 00:26:03
---

> Author: CubicYYY.
> Feel free to share, but please credit the source and include a link to the original article. Thanks! :)

## Intro

This is the first article in *Rust Async Demystified* series.

In this series, I will introduce fundamental concepts about asynchronous programming.
Then, we will discover the asynchronous infrastructures in Rust and other languages. We can extend our recognition of asynchronous programming by comparing the pros and cons of different asynchronous system designs.
Our final task is to build a minimum Rust asynchronous runtime from scratch!

I use JavaScript in this series as pseudo-codes (they may be executable, though).
So, it is fine if you are NOT familiar with JavaScript.

Now, let's start our journey with asynchronous programming.

---

### What is Asynchronous Programming: Understand Key Concepts

**"Async" stands for asynchronous.**
In general, asynchronous programming is a programming paradigm that allows a program to execute other tasks while waiting for an operation to complete rather than blocking or waiting for those operations to finish.

That's a mouthful. So here's an analogy for it.
Imagine that you are making the dinner for your family:

- **Parallelism**: If you are always busy doing something, you have nothing to complain about: the faster you are, the earlier you yield the dinner. Of course, you can ask other people to help you, with some extra cost of coordinating.
Yes, that is basically the exact solution for CPU-bounded tasks in programming: to utilize multi-core resources, we can introduce **parallelism** mechanisms like multi-threading or multi-processing, though at the cost of coordinating different threads/processes.

- **Concurrency**: the ability to make multiple tasks "seem like" they are running simultaneously. Asynchronous programming is one way, but not the only way, to achieve concurrency. For example, threading systems also provide concurrency even in a single-threaded environment. Parallelism is executing tasks physically simultaneously, so, of course, it is done with concurrency.

... But wait. What if you spend most of your time waiting? For example, you may always wait for the water to boil or the microwave to complete.
What will you do? Sit there doing nothing else, waiting for the water to boil since it is unfinished?
An ordinary adult will never make dinner like this. But this is precisely what a synchronous program does.

- **Synchronous programming**: executes sequentially and does nothing valuable when waiting for an I/O task to finish. In this situation, we say the task "blocks" the program. So, **synchronous** programming does things sequentially; hence, it is inefficient in I/O-bounded situations, where you spend most of the time waiting.

The solution is obvious to humans: when you are waiting for a task, do something else until the task is ready to continue. For example, you can fry some chicken while baking cookies: if the oven sounds "ding--!", you know you can continue processing those cookies (right now or after you finish frying chicken; it is up to you).

- **Asynchronous programming**: it switches between different tasks without "blocking" yourself when waiting for some tasks. This is the so-called **asynchronous** style.

About **threads**: A thread is the smallest unit of execution within a process in the OS.
So, if the kitchen is like your computer, a process is like a group of cooks for specific dishes; each cook is like a thread. If the process(group) only consists of a main thread(cook), this is called **single-threaded**; otherwise, it is **multi-threaded**.
However, **the physical resource(e.g., CPU cores) restricts your parallelism**. If you only have an oven in your kitchen, then only one dish can be baked, regardless of how many cooks you have.

![Analogy between a kitchen and a computer](kitchen-vs-computer.svg)

An oven (CPU core) can only be used by a single cook (thread).

Here's the conclusion:

- Task **execution styles**:
  - **Concurrency**: Multiple tasks seem to be managed "simultaneously" but not necessarily simultaneously. This is achieved via techniques like threading and asynchronous programming.
  - **Parallelism**: Task execution is physically simultaneous (really at the same time), so task concurrency naturally occurs. It requires multiple processors or cores.
- **Programming models**:
  - **Synchronous**: A blocking model in which tasks are performed one after another (sequentially). Typically, this involves blocking operations to wait for tasks to complete. In single-threaded environments, concurrency is impossible in synchronous codes.
  - **Asynchronous**: This is a non-blocking model in which Tasks can run other tasks without waiting for the current one to finish. It is often used to prevent blocking in operations like I/O. This is one way to obtain concurrency and even achieve parallelism in a multi-threaded environment.
- Task **execution environments**:
  - Single-threaded
  - Multi-threaded

Takeaways:

- I/O-bounded 👉 add concurrency 👉 use asynchronous programming to avoid idling in waiting
- CPU-bounded 👉 add parallelism 👉 introduce multi-threading / multi-processing to execute more calculations/instructions
- **Parallelism is a method to achieve concurrency, which is only available in a multi-threaded environment.**
- **Implementing concurrency can make a single-threaded system seem to have parallelism (but actually not).**

---

### Concurrency vs Parallelism

These are two concepts that often confuse people. Let's compare briefly.

First, **concurrency is not necessarily parallelism**, but **parallelism is always a way to achieve concurrency** (if parallelism is possible). For example, a single-core computer can achieve concurrency by switching in processes where they appear to be running at the same time. A multi-core computer, on the other hand, can physically execute multiple instructions or multiple computations at the same time, which is of course concurrent.

So, parallelism is a silver bullet? It seems like it always obtains concurrency and never blocks your program, right?
Unfortunately, no.

The problem is that concurrency is more than having or not having; it is also **about high and low concurrency**.
Indeed, parallelism can obtain concurrency for your program in a multi-threaded environment. However, asynchronous programming can still provide even higher concurrency.
On the other hand, asynchronous codes still **blocks in each thread**, even if parallelism is introduced into a synchronous program,.

Let's continue the analogy of making dinner in a kitchen.
Calling people to help you in the kitchen is to introduce parallelism.

- The **synchronous** way: each person executes their tasks sequentially. If most of them are waiting for their current task, the kitchen will be full of people waiting. Even worse, a person's current task may wait for another's task to finish, so parallelism is lost.
- The **asynchronous** way: Every person finds a task to process when waiting for a task. Suppose a person finishes all the tasks; then the person can even help others by doing other's tasks (this is called the *work-stealing policy*)! That is the efficient way to do things.

So here's a key point of asynchronous programming:

> Asynchronous programming is one way to achieve concurrency. It can also obtain parallelism in a multi-threaded environment.

Figures will make it much easier to understand. Suppose we have some tasks to do:

![tasks](tasks.svg)

- Sync + Single-Threaded Environment: it blocks.
   ![blocking](sync-single.svg)
- Async + Single-Threaded Environment: it switches between tasks to obtain concurrency.
   ![switching](async-single.svg)
- Sync + Parallelism: in-task parallelism, but still sequentially execute tasks.
   ![partial parallelism](sync-multi.svg)
  - Even some tasks may have internal parallelism, all tasks have to run sequentially.
- Async + Parallelism: full parallelism.
   ![full parallelism](async-multi.svg)
  - As soon as the current task completes or needs to wait, the thread is immediately idle and ready to receive a task that is ready to be executed.

---

### How Async Programming Works: The Underlying Mechanisms

Humans are born to act asynchronously (I hope so, especially for people with driving licenses).

However, this becomes tricky in programming: How does a computer "know" something happened and continue processing?
The key challenge is, for example, that after the program calls a function to issue an I/O request, it needs to be notified when the request is ready and react correctly to this.

Models/styles able to achieve this goal include:

- **Polling**: The program repeatedly checks the task status at regular intervals. However, it occupies the CPU and blocks the program. It's not good, but sometimes you have to, especially in synchronous codes.
- **Callbacks**: To avoid polling, we can directly tell the I/O function what it should do after the I/O operation is finished by passing a **callback**, which is something like a function pointer. Callback is a kind of asynchronous programming. Understanding callbacks can be the *foundation\** for learning asynchronous programming systems.
- **Event-Driven**: The program's flow is determined by events such as user actions, timer expirations, etc. An **event loop** repeatedly checks for events and dispatches them to appropriate handlers.
- ...

Here are some conceptual trivial discussions; you don't need them to write codes, but they are inspiring:

The use of the word *foundation* is apt here because many implementations of asynchronous programming can be seen as special cases of callbacks. Essentially, calling a function involves jumping to a specific location in memory, and callbacks specify where the function should jump to after completing a task. This can be explicit, as with a defined callback function, or implicit, as with a "callback location", which might be in the middle of a function.

The key takeaway is: **To react correctly, a program needs to know where to jump next**. Therefore, managing control flow is crucial in asynchronous programming.

> The "callback location" after calling a `wait()` function is the next line of code.
> In an event loop, the event dispatcher finds the appropriate handler-—or the callback—-for an event.
> Continuation-Passing Style (CPS) in functional programming can be implemented by passing a callback parameter to functions.

---

To clarify, though we can implement callbacks ourselves, they are possibly "synchronous callbacks": the function still blocks the program until it finishes its job, then calls the callback you passed in and returns. Synchronous callbacks may make your functions more flexible but will not increase concurrency.
On the other hand, "asynchronous callbacks" require involving some "magic" API functions in underlying libraries or provided by the operating system: down to the earth, it may register the program in some inner components, and the system will get notified when the operation is completed (via some mechanisms like interrupts or system event notifications), then trigger the callback you set (if any).
You can see many explicit asynchronous callbacks in JavaScript since it has a language built-in asynchronous runtime: it automatically finds the corresponding callback function (for each event) to call after complete synchronous codes in each [event loop](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Event_loop).

---

Languages like C/C++ and Rust, however, are not equipped with a built-in asynchronous runtime for [some reasons](https://stackoverflow.com/questions/66281348/why-does-cs-async-await-not-need-an-event-loop), so you can hardly see those callbacks (though they are possible).
In this case, many asynchronous I/O interfaces (e.g., `io_uring`) provided by the OS work like this: you issue an I/O request, which registers your program inside the system, and the program continues; when your program needs the result of that request to continue processing, just call a function like `wait()`. If the result is not yielded yet, the current thread cannot continue, so it will be parked and yield the CPU to other threads, so there is no blocking here; when the request is fulfilled, your program will be re-scheduled(woken up) by the OS to continue processing.
This is the event-driven style, and it utilizes interrupts or other low-level techniques to sense events effectively.

Is the callback gone here? Yes and no. A "general callback" is implicitly implemented in your system: the thread scheduler. Various events result in the same callback: wake the corresponding thread up. Then, your thread should check if it can really continue processing. If not, just go to sleep again (this is called a "spurious wakeup").

**Spoiler**: If the scheduler is implemented in the user space, you get [green threads](https://en.wikipedia.org/wiki/Green_thread), which belongs to asynchronous programming.

---

...We talked a lot about underlying mechanisms. Luckily, you don't need to be bothered with those trivial details! You, the API user, will probably get a "handle" immediately so you can continue doing other things (it is asynchronous).
You can check the current state of your requested tasks (fulfilled or not) manually via handles at any time. You can also use this handle to *wait* for a task: to *wait* for a task is to manually block the program until the task finishes.

![low-level mechanisms](low-level-async.svg)

---

### Why NOT to Use Callbacks Directly?

If a task has few steps, callbacks are good to go. For example, when you are in a cafe, you may get a buzzer from the barista after you order a coffee. He/she/they will tell you when the buzzer buzzes, "Come to get your coffee." Here, the barista sets a callback for you!
You can do something else, like browse this blog until the buzzer buzzes.
Ordering, waiting, getting your coffee, and leaving are the only steps in this task. Browsing the blog is a step in another task, and you can do it while you are waiting.

Here is a JavaScript to demonstrate it:

```javascript
// Define the callback function that will be called when the coffee is ready
function onCoffeeReady() {
    console.log("The buzzer buzzes! Come to get your coffee.");

    // Getting the coffee...

    console.log("You got your coffee. Enjoy!");
}

// Function to simulate ordering coffee
function orderCoffee(callback) {
    console.log("You order a coffee.");
    console.log("The barista gives you a buzzer.");
    console.log("You can do something else while waiting...");

    // Here comes the asynchronous part 
    setTimeout(() => {
        // When the coffee is ready, the callback is called
        callback();
 }, 3000); // Simulate a 3-second waiting time for coffee preparation
}

function visitCoffeeShop() {
    console.log("You enter the coffee shop.");
    
    // Order coffee and pass the callback for when it's ready
    orderCoffee(onCoffeeReady);

    // You are doing other things while waiting for the coffee
    console.log("You are browsing a blog while waiting...");
}

// Start the process
visitCoffeeShop();
```

However, more steps/phases in a task are common in programming. Sometimes, you even need to set multiple callbacks for a step (e.g., `onSuccess` and `onFail`).
For example, there are many stages in HTTP connection establishment, and transitions between states are A LOT.
In situations like this, callbacks start to nest into each other, and you can see the code becomes this:

```javascript
function step1(callback) {
    setTimeout(() => {
        console.log("Step 1 complete");
        callback();
 }, 1000); // Takes 1 second to complete
}

function step2(callback) {
    setTimeout(() => {
        console.log("Step 2 complete");
        callback();
 }, 1000); // Takes 1 second to complete
}

// step3, step4, step5, ...

// Execute the task:
step1(() => {
    step2(() => {
        step3(() => {
            step4(() => {
                console.log("Task completed.");
            });
        });
    });
});
// Oh no...
```

This pattern is called **callback hell**. It's almost impossible to maintain and extend. Callbacks lack scalability.
Directly writing all the callbacks yourself is terrible since the codes originally in a task are now spread into many callback functions, resulting in fragmented and unreadable codes.

So, it would be better not to write callbacks directly. Instead, we should wrap them in wrappers to program asynchronously more ergonomically. For example, we should use the `Promise` in JavaScript.

...or not to use explicit callbacks at all. **Most modern asynchronous APIs do not use callbacks.**
Modern asynchronous programming infrastructure provides human-friendly interfaces to programmers by abstraction, not raw callbacks.

---

## What's Next?

In the next article, we'll discover Rust's asynchronous infrastructure, and experience how it allows programmers to write asynchronous codes like synchronous codes.

---

## References

[Concurrency, Parallelism, Threads, Processes, Async, and Sync — Related? 🤔 | by G. Abhisek | Swift India | Medium](https://medium.com/swift-india/concurrency-parallelism-threads-processes-async-and-sync-related-39fd951bc61d)
[Introducing asynchronous JavaScript | MDN](https://developer.mozilla.org/en-US/docs/Learn/JavaScript/Asynchronous/Introducing)

Images are created using [Excalidraw](https://excalidraw.com/).

---

In *Rust Async Demystified* series:

- Part 1 - Basic Concepts of Async Programming
- Part 2 - Async Infrastructure in Rust and Other Languages (TODO)
- Part 3 - Build Yourself a Minimal Runtime from Scratch (TODO)

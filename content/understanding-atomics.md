+++ title = "Understanding Atomics" description = "Explains the basic of atomics in Rust. Comes with a bonus: spinlock implementation." date=2018-05-30 +++

# Abstract

In this post I will explain what are atomics and how to use them in rust.

# "What is an 'atomic' anyway?"

Atomics are data types on which operations are indivisible,
or the indivisible operations by themselves. Indivisible here means that
the side effects of the operations are only observable when the operation
is done. Generally, machines provide read and write operations over its
native integer both indivisible and divisible.

If you have ever heard of  "data races", divisible read and write causes them.
Suppose we are on a 64-bit word machine. Threads `A`, `B` and `C` have
access to address `p`. If `A` does a "divisible" write on `p` with `x`,
at the same time `C` does a "divisible" read, `C` might get half of previous
bits of `p` and half bits of `x`. Or a quarter of bits. Or the previous state.
Or the new state. It's actualy hard to predict. If thread `A` writes
"divisibily" again on `p` with `y` at the same time thread `B` writes
"divisibily" `z`, the resulting value of `p` might be a mix of both states or
something similar.

Because the behavior might vary between different machines, programming
languages such as Rust and C++ defines data races as undefined behavior. The
actual behavior might not be "leaving an intermediate value", but it could
be anything, from throwing an exception to setting the processor on fire.
You probably noticed that the problem here are the "divisible" operations.
If `A` had atomically written `x` on `p`, and `C` had atomically read from
`p`, `C` would see the bits of `p` either before of `A`'s write or after the
write, but not in the middle. The same apply for `A` and `B` writing
atomically; the final state would be either `y` or `z`, but not anything else.

# Atomics in Rust

Currently (rust nightly 1.28 and stable 1.26) these atomic data types were
stabilized: `AtomicPtr<T>`, `AtomicUsize`, `AtomicIsize` and `AtomicBool`.
They are all located in `core::sync::atomic` (or `std::sync::atomic`).
To explain, I will pick the simplest one: `AtomicBool`.

## `AtomicBool`

It is very simple to create one:

```rust
let atomic_bool = AtomicBool::new(true);
```

Now let's make an atomic read. But wait... uh oh... `AtomicBool::load`
accepts two arguments: san immutable reference to `self` and another
argument of type `Ordering`. The reference to `self` is easily understandable
but what about `Ordering`? `Ordering` is a type which determinates the...
the... order? Yes, the order related to other operations and other threads.
Let's see the definition of `Ordering`:

```rust
pub enum Ordering {
    Relaxed,
    Release,
    Acquire,
    AcqRel,
    SeqCst,
    // some variants omitted
}
```

There are, roughly speaking, two kinds of CPUs related to orderings: the weak
and the strong. The weak processors have naturally weak guarantees on the
orderings, while the strong processors have naturally strong guarantees on the
orderings. It is generally low-cost for strong-processors to perform `Acquire`,
`Relase` and `AcqRel`, while they are high-cost for weak-processors. For all
of them `Relaxed` is low-cost and `SeqCst` is high-cost. Examples of weak-
processors are ARM-archs, and examples of strong-processors are the ones
x86/x86_64-archs.

The only valid variants for `load` to be called with are: `Relaxed`, `Acquire`
and `SeqCst`. Don't worry, the other variants will be explained later. `Relaxed`
is somewhat like "the order does not matter at all". This allows both
the compiler and the CPU to reorder the operation. `SeqCst` means "sequentially
consistent"; it should not be reordered at all! Everything before it happens
before it, and everything after it happens after it.

`Acquire` is a bit more complex. It is the "complement" of `Release`.
Everything (generally `store`s) that happens after the `Acquire` stays after
it. But the compiler and the CPU are free to reorder anything that happens
before it. It is designed to be used with `load`-like operations when acquiring
locks.

Let's see an example with `AtomicBool::load`:

```rust
use std::sync::atomic::{
    AtomicBool,
    Ordering::*,
};

fn print_if_it_is(atomic: &AtomicBool) {
    if atomic.load(Acquire) {
        println!("It is!");
    }
}
```

Let's jump into the next operation: `store`. `store` accepts a reference to
`self`, the new value (of type `bool`), and an `Ordering`. The valid
`Ordering`s are `Relaxed`, `Release` and `SeqCst`. `Release`, as said before,
is used as a pair with `Acquire`. Everything before `Release` it happens before
it, but the compiler and the CPU are free to reorder anything that happens
after it. It is intended to be used when releasing a lock.

Let's see an example:

```rust
use std::sync::atomic::{
    AtomicBool,
    Ordering::*,
};

fn store_something(atomic: &AtomicBool) {
    atomic.store(false, Release);
}
```

You may have noticed two things. If you did not notice them, do it now. First:
`store` just needs an immutable reference. This means that it does not need
"outer" mutability, but it has inner mutability. Second: atomics are only
useful with shared references. They don't  have anything like "`clone`"; to
clone you can do somewhat `let copied = AtomicBool::new(src.load(Acquire));`.
But this does not share the atomic. Therefore, it is a common pattern to
encapsulate the atomic (or the struct which keeps the atomic) into an `Arc`.

There are at least three more important operations `swap`, `compare_and_swap`,
`compare_exchange_*`. Swap is pretty simple: it swaps its argument (of type
`bool`) with the stored value, and returns the stored value. `Ordering` on
swap accepts any of the `Ordering`s. `compare_and_swap` takes a `bool`, say,
"current", another `bool`, say "new", and the `Ordering` (which can be any).
`compare_and_swap` is a really important atomic operation. It compares the
argument "current" with the stored value. If they are the same, the operation
swaps the stored value with "new". In any case, the stored value is returned.
The operation succeeds if the return value is equal to the "current" argument.

There is also `compare_exchange_strong` (just `compare_exchange` in Rust), and
`compare_exchange_weak` (which may fail more often). They are similar to
`compare_and_swap`, except that they take two orderings: one for success, and
one for failure. Also, in Rust they return a `Result`. The fact that these
operations return a `Result` allows `compare_exchange_weak` to fail even if
the comparison succeed.

Wait! I did not explain `Ordering::AcqRel`. It is what it seems: it combines
`Acquire` when loading and `Release` when storing for operations that
load/store atomically. Although I explained all of these using `AtomicBool`,
they are pretty much common primitive operations for all primitive atomic data
types. All atomic data types in Rust have at least these operations: `load`,
`store`, `swap`, `compare_and_swap`, `compare_exchange`,
`compare_exchange_weak`. Enough of talk, let's act!

## Lock-free logical AND

The function below receives an atomic boolean, an ordinary boolean, stores the
logical AND of them and returns the previous value.

```rust
use std::sync::atomic::{AtomicBool, Ordering::{*, self}};

fn lockfree_and(x: &AtomicBool, y: bool, ord: Ordering) -> bool {
    let mut stored = x.load(match ord {
        AcqRel => Acquire,
        Release => Relaxed,
        o => o,
    });
    loop {
        let inner = x.compare_and_swap(stored, stored & y, ord);
        if inner == stored {
            break inner;
        }
        stored = inner;
    }
}
```

Note that side-effects are only visible after we're done. This is in, some way,
"atomic". Generally, this kind of operation is called "lock-free" because it
does not use any kind of lock. Although we have a loop, we do not depend on
any thread "free-ing" the resource. This is not a lock.

And, well... Rust already provides us a method `AtomicBool::fetch_and`, which
probably is translated into a single native instruction. I just wanted to show
you that with the primitives `load` and `compare_and_swap`, the operation can
be implemented via software.
[Take a look here](https://doc.rust-lang.org/std/sync/atomic/struct.AtomicBool.html#method.fetch_and),
we have other methods on `AtomicBool`, such as `nand`, `or`, and `xor`.

As you may suspect, `AtomicUsize` and `AtomicIsize` have their own methods with
basic arithmetic (add and sub) and bitwise. And all of them are implementable
via software, but probably translated into a single native instruction.
Although the API does not provide `mul` and `div`, it is not hard to implement
them.

## A note on `AtomicPtr<T>`

`AtomicPtr<T>` can be confusing. The methods `load` and `store` read and write
the address or its content? The answer is: the address. They operate on `*mut T`.
To load the contents is pretty much another operation, but it is not atomic and
and it is very unsafe.

## `Send` and `Sync`

You may have heard of the auto traits `Send` and `Sync`. They matter a lot in a
concurrent environment. `T: Send` means `T` is safe to be sent to another
thread. `T: Sync` means `&T` is safe to be shared between threads. Atomic data
types implement both `Send` and `Sync`, so it is safe to share and send them
among threads.


## Bonus: Implementing a spinlock

```rust
pub mod spinlock {
    use std::{
        cell::UnsafeCell,
        fmt,
        ops::{Deref, DerefMut},
        sync::atomic::{
            AtomicBool,
            Ordering::*,
        },
    };

    #[derive(Debug)]
    pub struct Mutex<T> {
        locked: AtomicBool,
        inner: UnsafeCell<T>,
    }

    #[derive(Debug, Clone, Copy)]
    pub struct MutexErr;

    pub struct MutexGuard<'a, T>
    where
        T: 'a
    {
        mutex: &'a Mutex<T>,
    }

    impl<T> Mutex<T> {
        pub fn new(data: T) -> Self {
            Self {
                locked: AtomicBool::new(false),
                inner: UnsafeCell::new(data),
            }
        }

        pub fn try_lock<'a>(&'a self) -> Result<MutexGuard<'a, T>, MutexErr> {
            if self.locked.swap(true, Acquire) {
                Err(MutexErr)
            } else {
                Ok(MutexGuard {
                    mutex: self,
                })
            }
        }

        pub fn lock<'a>(&'a self) -> MutexGuard<'a, T> {
            loop {
                if let Ok(m) = self.try_lock() {
                    break m;
                }
            }
        }
    }

    unsafe impl<T> Send for Mutex<T>
    where
        T: Send,
    {
    }

    unsafe impl<T> Sync for Mutex<T>
    where
        T: Send,
    {
    }

    impl<T> Drop for Mutex<T> {
        fn drop(&mut self) {
            unsafe {
                self.inner.get().drop_in_place()
            }
        }
    }

    impl<'a, T> Deref for MutexGuard<'a, T> {
        type Target = T;

        fn deref(&self) -> &T {
            unsafe {
                &*self.mutex.inner.get()
            }
        }
    }

    impl<'a, T> DerefMut for MutexGuard<'a, T> {
        fn deref_mut(&mut self) -> &mut T {
            unsafe {
                &mut *self.mutex.inner.get()
            }
        }
    }

    impl<'a, T> fmt::Debug for MutexGuard<'a, T>
    where
        T: fmt::Debug,
    {
        fn fmt(&self, fmtr: &mut fmt::Formatter) -> fmt::Result {
            write!(fmtr, "{:?}", &**self)
        }
    }

    impl<'a, T> Drop for MutexGuard<'a, T> {
        fn drop(&mut self) {
            let _prev = self.mutex.locked.swap(false, Release);
            debug_assert!(_prev);
        }
    }

}
```

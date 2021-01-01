---
layout: post
title: An unsafe tour of Rust's Send and Sync
date: 2021-01-01 06:54 -0800
---

Rust's concurrency safety is based around the `Send` and `Sync` traits. For people writing safe code, you don't really need to understand these traits on a deep level, only enough to satisfy the compiler when it spits errors at you (or switch from `std` threads to Crossbeam scoped threads to make errors go away). However if you're writing unsafe concurrent code, such as having a `&UnsafeCell<T>` hand out `&T` and `&mut T`, you need to understand `Send` and `Sync` at a more fundamental level, to pick the appropriate trait bounds when writing `unsafe impl Send/Sync` statements, or add the appropriate `PhantomData<T>` to your types.

In this article, I will explore the precise behavior of `Send` and `Sync`, and explain *why* the standard library's trait bounds are the way they are.

## Prior art

> You can think of Send as "Exclusive access is thread-safe," and Sync as "Shared access is thread-safe."
>
> [[Source]](https://www.reddit.com/r/rust/comments/9elom2/why_does_implt_send_for_mut_t_require_t_send/)

I recommended first reading ["Rust: A unique perspective"](https://limpet.net/mbrubeck/2019/02/07/rust-a-unique-perspective.html). This article gives a conceptual overview of the mechanics (unique and shared references) I will analyze in more depth.

## Defining Sync and Send

`T: Send` means `T` and `&mut T` (which allow dropping `T`) can be passed between threads. `T: Sync` means `&T` (which allows shared/aliased access to `T`) can be passed between threads. Either or both may be true for any given type. `T: Sync` ≡ `&T: Send` (by definition).

One way that `T: !Sync` can occur is **if a type has non-atomic interior mutability**. This means that every `&T` (there can be more than one) can mutate `T` at the same time non-atomically, causing data races if a `&T` is sent to another thread. This includes `Cell<T>` and `RefCell<T>`, as well as `Rc<T>` (which points to a `Cell<RefCount>`).

`T: !Send` **if a type is bound to the current thread**. Examples:

- `MutexGuard`, where the "unlock" syscall must occur on the same thread as "lock".
- `&V` where `V` can be modified non-atomically (only safe from a single thread) through multiple `&V` (explained above).

## Primitives

Most primitive types (like `i32`) are `Send+Sync`. They can be read through shared references (`&`) by multiple threads at once (`Sync`), and modified through unique references (`&mut`) by any one thread at a time (`Send`).

## Owning references

`Box<T>` and `&mut T` give the same access as having a `T` directly, so it shares the same Sync/Send status as `T`.

(Sidenote) Technically, `&mut T` allows swapping the `T` (which cannot panic), but prohibits moving the `T`. This is because moving invalidates the `&mut T`, and the `&mut T`s and `T` it's constructed from.

For a demonstration of `&mut T`, see ["Example: Passing `&mut (T: Send)` between threads"](#example-passing-mut-t-send-between-threads) section in this page.

### Where these semantics are defined

- [`impl Send for &mut T where T: Send`](https://doc.rust-lang.org/std/primitive.reference.html#impl-Send-1)
- `impl Sync for &mut T where T: Sync` is not in the page...
- [`impl Send for Box<T> where T: Send`](https://doc.rust-lang.org/std/boxed/struct.Box.html#impl-Send)
- [`impl Sync for Box<T> where T: Sync`](https://doc.rust-lang.org/std/boxed/struct.Box.html#impl-Sync)

## Shared references

Unlike owning references, many `&T` can be created from the same `T`. And an unlimited number of `&T` and `Rc<T>` and `Arc<T>` copies/clones can point to the same `T`.

By definition, you can `Send` `&T` instances to other threads iff `T` is `Sync`. For example, `&i32` is `Send` because `i32` is `Sync`.

Less obvious is that `&T: Sync` requires that `T: Sync`. Why is this the case?

- Why must `T` be `Sync`? If we want `&T` to be `Sync`. This means `&&T` (which is clonable/copyable) is `Send`, allowing multiple threads to concurrently obtain `&&T` and `&T`, which is only legal if `T` is `Sync`.
- Why is `&&T: Send` legal? Because `&T` lacks interior mutability (a `&&T` can't modify the `&T` to point to a different `T`).

### Sources

- [`impl Send for &T where T: Sync`](https://doc.rust-lang.org/std/primitive.reference.html#impl-Send)
- `impl Sync for &T where T: Sync` is not in the page...
  - For a demonstration, see the ["Example: `&T: Send or Sync` both depend on `T: Sync`"](#example-t-send-or-sync-both-depend-on-t-sync) section in this page.

## Interior mutability

`Cell<i32>` (and `RefCell<i32>`) is `!Sync` because it has single-threaded **interior mutability**, which translates to multithreaded **data races**.

`UnsafeCell<i32>` is `!Sync` to prevent misuse, since only some usages are `Sync` and `impl !Sync` is unstable. As a result, you need to `unsafe impl Sync` (which shows up in grep) if you want concurrent access.

## Smart pointers: `Rc<T>`

`Rc<i32>` acts like `&(Cell<RefCount>, i32)`. It is `!Sync` because `Cell<RefCount>` has **interior mutability** and **data races** on `RefCount`, and `!Send` because `Rc<T>` is clonable, acts like a`&Cell<RefCount>`, and `Cell<RefCount>` is `!Sync`.

(Technically `Rc<i32>` also acts like `&mut T` in its ability to drop `T`, but it doesn't matter because it's always `!Send` and `!Sync`.)

### Sources

- [`impl<T> !Send for Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html#impl-Send)
- [`impl<T> !Sync for Rc<T>`](https://doc.rust-lang.org/std/rc/struct.Rc.html#impl-Sync)

## Smart pointers: `Arc<T>` (atomic refcounting)

`Arc<T>` is a doozy. It acts like `&(Atomic<RefCount>, T)` in its ability to alias `T`, and `T`/`&mut T` in its ability to drop or `get_mut` or `try_unwrap` the `T`.

Because `&T` can alias, `Arc<T>: Send+Sync` requires `T: Sync`.

Additionally, `Arc<T>: Send` requires `T: Send` (because you can move `Arc<T>` across threads, and `T` with it).

And `Arc<T>: Sync` requires `T: Send`, because if `T: !Send` but `Arc<T>: Sync`, you could clone the Arc (via `&Arc<T>`) from another thread, and drop (or `get_mut` or `try_unwrap`) the clone last, violating `T: !Send`.

(`Atomic<RefCount>` is `Send+Sync` and does not contribute to `Arc`'s thread safety.)

### Sources

- [`impl<T> Send for Arc<T> where T: Send + Sync`](https://doc.rust-lang.org/std/sync/struct.Arc.html#impl-Send)
- [`impl<T> Sync for Arc<T> where T: Send + Sync`](https://doc.rust-lang.org/std/sync/struct.Arc.html#impl-Sync)

This was also discussed in a [Stack Overflow question](https://stackoverflow.com/questions/41909811/why-does-arct-require-t-to-be-both-send-and-sync-in-order-to-be-send).

## Mutexes

`Mutex<T>` is `Sync` even if `T` isn't, because if multiple threads obtain `&Mutex<T>`, they can't all obtain `&T`.

`Mutex<T>: Sync` requires `T: Send`. We want `&Mutex` to be `Send`, meaning multiple threads can lock the mutex and obtain a `&mut T` (which lets you swap `T` and control which thread calls `Drop`). To hand-wave, exclusive access to `T` gets passed between threads, requiring that `T: Send`.

`Mutex<T>: Send` requires `T: Send` because `Mutex` is a value type.

`MutexGuard<T>` is `!Send` because it's **bound to the constructing thread** (on some OSes including Windows, you can't send or exchange "responsibility for freeing a mutex" to another thread). Otherwise it acts like a `&mut T`, which is `Sync` if T is `Sync`. Additionally you can extract a `&mut T` (which is `Send`) using `&mut *guard`.

### Sources

- [`Mutex` traits](https://doc.rust-lang.org/std/sync/struct.Mutex.html#impl-Send)
- [`MutexGuard` traits](https://doc.rust-lang.org/std/sync/struct.MutexGuard.html#impl-Send)

### Contrived corner cases

`Mutex<MutexGuard<i32>>` is `!Sync` because `MutexGuard<i32>` is `!Send`.

## Thoughts on trait bounds and flexibility for users

Why does `Arc<T>` not have a `where T: Send + Sync` trait bound, but instead allows you to construct `Arc<T>` for any `T` (but just not send/share it across threads)?

I've heard that you should avoid putting trait bounds in types, but (if I remember correctly) instead in method implementations, or (in the case of `Arc`) in conditional `Send`/`Sync` implementations. One person said:

> The reason the restrictions are usually on the implementations rather than on the type in general is that you don't usually know every possible implementation
> If you later realize you can add other functionality, you can just add additional impl blocks with different restrictions, whereas if they were on the type you would potentially have to worry about unifying the restrictions (which can be really awkward) or removing them altogether

When asking about this topic, I was pointed to the [Rust API guidelines](https://rust-lang.github.io/api-guidelines/about.html), but I couldn't find any discussion of this issue.

----

I personally encountered this topic when I used an `Arc` internally for [the `flip-cell` crate](https://github.com/nyanpasu64/spectro2/blob/master/flip-cell/src/lib.rs) (which turns out to be equivalent to [Oddio's `Swap` type](https://github.com/Ralith/oddio/blob/main/src/swap.rs) and the [`triple-buffer` crate](https://github.com/HadrienG2/triple-buffer)).

`Arc<T>: Sync` is only safe if `T: Send`, not just `T: Sync`; this is because another thread can look at an `&Arc<T>`, clone it, and obtain an `Arc<T>` sharing ownership over the same object. But if we create a type `FlipReader<T>` ([source](https://github.com/nyanpasu64/spectro2/blob/05561a21d85fc5fc0e8e92140edf01d6b64401bc/flip-cell/src/lib.rs#L188-L201)) which contains an `Arc<Wrapper<T>>` but prohibits cloning it, then making `FlipReader<T>: Sync` does not allow another thread to take shared ownership of `Wrapper<T>`, so the `Wrapper<T>: Send` trait bound is unnecessary.

Had the struct `Arc<T>` required `T: Send + Sync` to even be constructed, `Arc` would be crippled as a building block for unsafe code.

## Example: Passing `&mut (T: Send)` between threads

Cell is `Send` but not `Sync`. Both `Cell` and `&mut Cell` can be passed between threads. The following code builds as-is, but not if `&mut` is changed to `&`.

```rust
use std::thread;
use std::cell::Cell;

fn main() {
    // Send + !Sync
    let cell_ref: &mut Cell<i32> = Box::leak(Box::new(Cell::new(0)));

    thread::spawn(move || {
        cell_ref.replace(1);
    });
}
```

## Example: `&T: Send or Sync` both depend on `T: Sync`

If `T: !Sync` (for example `Cell`), then `&T` is neither `Send` nor `Sync`.

```rust
use std::cell::Cell;

fn ensure_sync<T: Sync>(_: T) {}
fn ensure_send<T: Send>(_: T) {}

fn main() {
    let foo = Cell::new(1i32);
    ensure_sync(&foo);
    ensure_send(&foo);
}
```

Trying to compile this code returns the errors:

```
Standard Error

   Compiling playground v0.0.1 (/playground)
error[E0277]: `Cell<i32>` cannot be shared between threads safely
 --> src/main.rs:8:17
  |
3 | fn ensure_sync<T: Sync>(_: T) {}
  |                   ---- required by this bound in `ensure_sync`
...
8 |     ensure_sync(&foo);
  |                 ^^^^ `Cell<i32>` cannot be shared between threads safely
  |
  = help: within `&Cell<i32>`, the trait `Sync` is not implemented for `Cell<i32>`
  = note: required because it appears within the type `&Cell<i32>`

error[E0277]: `Cell<i32>` cannot be shared between threads safely
 --> src/main.rs:9:17
  |
4 | fn ensure_send<T: Send>(_: T) {}
  |                   ---- required by this bound in `ensure_send`
...
9 |     ensure_send(&foo);
  |                 ^^^^ `Cell<i32>` cannot be shared between threads safely
  |
  = help: the trait `Sync` is not implemented for `Cell<i32>`
  = note: required because of the requirements on the impl of `Send` for `&Cell<i32>`
```

If `T: !Send + Sync` (for example `MutexGuard`), then `&T` is still `Send + Sync`. (This makes sense, because `T: !Send` only constrains the behavior of a `&mut T`, and should not affect the properties of a `&T`.)

```rust
use std::marker::PhantomData;
use std::sync::MutexGuard;

fn ensure_sync<T: Sync>(_: T) {}
fn ensure_send<T: Send>(_: T) {}

fn main() {
    let foo = PhantomData::<MutexGuard<i32>> {};
    ensure_sync(&foo);
    ensure_send(&foo);
}
```

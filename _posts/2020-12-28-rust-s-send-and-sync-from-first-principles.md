---
layout: post
title: Rust's Send and Sync from first principles
date: 2020-12-28 04:20 -0800
---

## Prior art

> You can think of Send as "Exclusive access is thread-safe," and Sync as "Shared access is thread-safe."
>
> [[Source]](https://www.reddit.com/r/rust/comments/9elom2/why_does_implt_send_for_mut_t_require_t_send/)

Recommended reading: ["Rust: A unique perspective"](https://limpet.net/mbrubeck/2019/02/07/rust-a-unique-perspective.html) gives a conceptual overview of the mechanics (unique vs. shared references) I will analyze in more depth.

## Defining Sync and Send

`T: Send` means `T` and `&mut T` (which allow dropping `T`) can be passed between threads. `T: Sync` means `&T` (which allows shared/aliased access to `T`) can be passed between threads. Either or both may be true for any given type. `T: Sync` ≡ `&T: Send` (by definition).

One way that `T: !Sync` can occur is **if a type has non-atomic interior mutability**, allowing `&T` to mutate `T`, causing data races if `&T` is sent to another thread.

`T: !Send` **if a type is bound to the current thread**. Examples:

- `MutexGuard`, where the "unlock" syscall must occur on the same thread as "lock".
- `&V` where `V` can be modified non-atomically (only safe from a single thread) through multiple `&V`. This includes `Cell<T>` and `RefCell<T>`.
  - This also applies to `Rc<T>` which holds a `Cell<RefCount>` internally.

## Primitives

Most primitive types (like `i32`) are `Send+Sync`.

## Owning references

`Box<T>` and `&mut T` give the same access as having a `T` directly, so it shares the same Sync/Send status as `T`.

Sidenote: technically, `&mut T` allows swapping the `T` (which cannot panic), but prohibits moving the `T`. This is because moving invalidates the `&mut T`, and the `&mut T`s and `T` it's constructed from.

### Where these semantics are defined

- [`impl Send for &mut T where T: Send`](https://doc.rust-lang.org/std/primitive.reference.html#impl-Send-1)
- `impl Sync for &mut T where T: Sync` is not in the page...
- [`impl Send for Box<T> where T: Send`](https://doc.rust-lang.org/std/boxed/struct.Box.html#impl-Send)
- [`impl Sync for Box<T> where T: Sync`](https://doc.rust-lang.org/std/boxed/struct.Box.html#impl-Sync)

## Shared references

Unlike owning references, many `&T` can be created from the same `T`. And an unlimited number of `&T` and `Rc<T>` and `Arc<T>` copies/clones can point to the same `T`.

By definition, you can `Send` `&T` instances to other threads iff `T` is `Sync`. For example, `&i32` is `Send` because `i32` is `Sync`.

Less obvious is that `&i32` is `Sync` iff `i32` is `Sync`. Why is this the case?

- Why must `i32` be `Sync`? We want `&i32` to be `Sync`. This means `&&i32` (which is clonable/copyable) is `Send`, allowing multiple threads to concurrently obtain `&&i32` and `&i32`, which is only legal if `i32` is `Sync`.
- Why is `&&i32: Send` legal? Because `&T` lacks interior mutability (a `&&T` can't modify the `&T` to point to a different `T`).

### Sources

- [`impl Send for &T where T: Sync`](https://doc.rust-lang.org/std/primitive.reference.html#impl-Send)
- `impl Sync for &T where T: Sync` is not in the page...

## Interior mutability

`Cell<i32>` (and `RefCell<i32>`) is `!Sync` because it has single-threaded **interior mutability**, which translates to multithreaded **data races**.

`UnsafeCell<i32>` is `!Sync` to prevent misuse, since only some usages are `Sync` and `impl !Sync` is unstable. As a result, you need to `unsafe impl Sync` (which shows up in grep) if you want concurrent access.

## Smart pointers

`Rc<i32>` acts like `&(Cell<RefCount>, i32)`. It is `!Sync` because `Cell<RefCount>` has **interior mutability** and **data races** on `RefCount`, and `!Send` because `Rc<T>` is clonable, acts like a`&Cell<RefCount>`, and `Cell<RefCount>` is `!Sync`.

(Technically `Rc<i32>` also acts like `&mut T` in its ability to drop `T`, but it doesn't matter because it's always `!Send` and `!Sync`.)

----

`Arc<T>` is a doozy. It acts like `&(Atomic<RefCount>, T)` in its ability to alias `T`, and `T`/`&mut T` in its ability to drop or `get_mut` or `try_unwrap` `T`.

Because `&T` can alias, `Arc<T>: Send+Sync` requires `T: Sync`.

Additionally, `Arc<T>: Send` requires `T: Send` (because you can move `Arc<T>` across threads, and `T` with it).

And `Arc<T>: Sync` requires `T: Send`, because if `T: !Send` but `Arc<T>: Sync`, you could clone the Arc (via `&Arc<T>`) from another thread, and drop (or `get_mut` or `try_unwrap`) the clone last, violating `T: !Send`.

(`Atomic<RefCount>` is `Send+Sync` and does not contribute to `Arc`'s thread safety.)

## Mutexes

`Mutex<T>` is `Sync` even if `T` isn't, because if multiple threads obtain `&Mutex<T>`, they can't all obtain `&T`.

`Mutex<T>: Sync` requires `T: Send`. We want `&Mutex` to be `Send`, meaning multiple threads can lock the mutex and obtain a `&mut T` (which lets you swap `T` and control which thread calls `Drop`). To hand-wave, exclusive access to `T` gets passed between threads, requiring that `T: Send`.

`Mutex<T>: Send` requires `T: Send` because `Mutex` is a value type.

`MutexGuard<T>` is `!Send` because it's **bound to the constructing thread** (on some OSes, you can't send or exchange "responsibility for freeing a mutex" to another thread). Otherwise it acts like a `&mut T`, which is `Sync` if T is `Sync`. Additionally you can extract a `&mut T` (which is `Send`) using `&mut *guard`.

### Contrived corner cases

`Mutex<MutexGuard<i32>>` is `!Sync` because `MutexGuard<i32>` is `!Send`.

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

---
layout: post
title: The gridline mental model of indexing and slicing
date: 2020-05-08 13:26 -0700
---

*Republished from my [Github gist](https://gist.github.com/nyanpasu64/c01e50ad97b1a92ccea374c3f941dd93#file-index-md).*

Integer indexes can either represent fenceposts (gridlines) or item pointers, and there's a sort of duality.

## Mental model: Gridline-based "asymmetric indexing"

Memory or data is treated as a "pool of memory". Pointers and indices do not refer to *elements*, but *gaps between elements* (in other words, fenceposts or gridlines). This is the same way I think about wall clocks and musical time subdivision, where time is continuous and timestamps refer to *instants* which separate regions of time.

In C and Python, array indexing can be interpreted via a mental model of gridlines. If `a` is an array holding elements, then `a[x]` is the element after gridline `x`. I call this "asymmetric indexing" (since every pointer refers to memory lying on the right side of it), but it's a useful convention. In C, if the array `a` holds elements of size `s`, `a[x]` occupies bytes from `(byte*)(a) + s*x` up until `(byte*)(a) + s*(x+1)`.

In Python, if `x` is a list (actually a resizable contiguous array), `x[0]` is the first element (after gridline 0), and `x[-1]` is the last element, 1 before the end (after gridline `len(x)-1`). This behavior matches a subset of modular arithmetic.

In C++, `iterator` and `reverse_iterator` both point to fenceposts between items. An array can have valid iterators or reverse iterators pointing to "before the first element", "after the last element", or anywhere in between.

Dereferencing a forward iterator accesses the element *after* the gridline, much like `*ptr` with a raw pointer. However, dereferencing a reverse iterator accesses the element *before* the gridline, which compiles to `*(ptr - 1)`. As a result, `reverse_iterator` appears to be slightly slower on actual CPUs: <https://stackoverflow.com/a/2549554>.

cppreference.com has a diagram attempting to explain `reverse_iterator`:

![cppreference.com reverse_iterator diagram](http://upload.cppreference.com/mwiki/images/3/39/range-rbegin-rend.svg)

I think the diagram is badly designed and unnecessarily confusing, with two arrows coming from the top of the diagram, and two pictures of the array offset by one element. It's technically not wrong, but it assumes that pointers point to *objects*, not *fenceposts*, which is a very inelegant mental model for this purpose.

## Alternative mental model: Item-based indexing

In pure math and DSP, and at high levels of abstraction, you can instead treat each item as an indivisible entity, rather than occupying a region of memory bounded between 2 endpoints. Then indexing points to an object, not an address or gridline in memory. In this mental model, slicing behaves quite differently.

You can choose to index from 0 or 1. Indexing from 0 or 1 is somewhat orthogonal to gridline-based or item-based indexing. However, most gridline-based languages index from 0, and many item-based languages index from 1.

The R language operates under this mental model. Much like mathematical notation, indexes begin at 1, and ranges of items `a[1:5]` are inclusive on both ends. In fact, `1:5` generates a vector of integers `1 2 3 4 5`.

The item-based mental model (with inclusive ranges) is useful in some cases, for example in DSP. However I moved that to a separate article, ["Describing convolution using item-based indexing and inclusive ranges"](../describing-convolution-using-item-based-indexing-and-inclusive-ranges), since it's not closely related to indexing.

## Gridline-based slicing and closed-closed indexing

Assume you have an array `a` with `N` elements.

For a region between gridlines `a ≤ b` to be valid, `a ≥ arr` and `b ≤ arr + N`. Note that `b` (and also `a`) is allowed to be equal to the final gridline, which is a perfectly valid gridline! The only reason people consider it "out of bounds" or "past the end of the array" is because it has no element to its right (cannot be used for asymmetric indexing).

What are the valid indices into an array of length N, treating the first element as 0?

- Conventional wisdom believes that valid array indices lie in a closed-open range.
- begin ∈ [0..N) since element 0 is valid, but element 0 is past the end of the array.

Another approach is to model "valid array indices" as a special case of "valid array slices", where the slice is of length 1. Under this approach, valid indices lie within a "closed-closed" inclusive range.

What are the valid starting indices of length-1 regions within an array?

- begin ≥ 0, otherwise the start of the region will lie outside the array.
- begin + 1 ≤ N, otherwise the end of the region will lie outside the array.
- begin ∈ [0..N-1]

What are the valid starting indices of length-2 regions within an array?

- begin ≥ 0, otherwise the start of the region will lie outside the array.
- begin + 2 ≤ N, otherwise the end of the region will lie outside the array.
- begin ∈ [0..N-2]

In summary, obtaining indexes from inclusive ranges are good because "valid indexes" are a special case of "valid slice starting indices" which are modeled well by inclusive ranges. Under this line of logic, a pair of pointers, `(pointer to begin, pointer to end)`, describes a slice of memory. I feel like this mental model is underused, and explaining it would help people understand C++'s `reverse_iterator` better.

Obtaining indexes from half-open ranges are good if you either assume "asymmetric indexing" (don't think in terms of slicing), or treat each item as an indivisible entity (alternative mental model, zero-indexed). Under this line of logic, `(pointer to begin, pointer to end)` is interpreted as `(pointer to first element, pointer past the final element)`, which is how how I've seen it be described by some people.

I feel languages should have both half-open ranges to generate indexes, and inclusive ranges to generate slice endpoints. Python only has half-open ranges, and math only has inclusive ranges. Rust has both, but unfortunately inclusive ranges are very slow and unoptimized compared to half-open ranges.

## Issue: negative indexing is asymmetric

To me, negative indexing is awkward in python. The first 2 elements in an list are `a[0]` and `a[1]`, but the last 2 elements are `a[-1]` and `a[-2]`. Interpreting this under the grid model, this arises because indexing `a[i]` takes the element *after* gridline `i`, which is inherently asymmetric.

## Issue: modular negative slicing and circularity is ambiguous

In Python, *item indexes* into a length-N array (where integer indices refer to the item after the gridline) conform to mod-N arithmetic. Each integer index is either interpreted mod N, or raises an "out of bounds" exception.

However, *slice endpoints* do *not* quite conform to modular indexing mod N. This is because fenceposts 0 and N are distinct gridlines in memory, but are conflated under mod-N operation.

In Python, if you want to access the last 2 elements in a length-N array, you can write `a[N-2:N-0]`. If you treat slice endpoints as modular indexes mod N, you can abbreviate this to `[-2:-0]`. But this instead returns an empty slice from N-2 to 0, since unlike array indexes, Python slice endpoints don't quite conform to modular indexing. And Python has no concept of a "negative zero" integer index meaning something different.

Because CSS grid has no fencepost 0, it sidesteps this issue entirely. Negating a slice endpoint always switches between "counting from the left" and "counting from the right".

### Numpy violates modular arithmetic

One place where this issue comes up is in Numpy. By analogy, in Python, you can assign to list slices to replace part of a list with other elements. For example, you can write `a[x:y] = [...]`. To insert one item, you can write `a[x:x] = [1]`. Given Python's slicing rules, `a[0:0] = [1]` inserts an element before the beginning of the list, and `a[-1:-1] = [1]` inserts an element *before* the last element of the list (not at the end of the list!) This is better written as `a.insert(x, 1)` where x can be any valid fencepost (including N which is not a valid index).

Numpy has an operation called `np.stack()` where you combine two or more N-dimensional arrays into a N+1-dimensional array. All input arrays have identical `shape: N-dimensional tuple` determining the dimensionality needed to index the array all the way. The output array has the same `shape` as the inputs, but with an extra element equal to the number of arrays you've passed in.

`np.stack(axis=0)` is analogous to `shape.insert(0, number of inputs)`. But `np.stack(axis=-1)` is analogous to `shape.insert(N - 0, number of inputs)`, not `N - 1`. 🤢

### CSS Grid fixes negative slicing but not negative indexing

CSS Grid allows web developers to dynamically position elements in table-like grids. In this case, fenceposts are *literally* gridlines between on-screen items. A layout with N columns (declared using `grid-template-columns`) has N+1 gridlines. (Tracks are columns or rows.)

Interestingly, gridline 0 does not exist. Gridline 1 is the leftmost gridline before the first item, and gridline N+1 is the rightmost gridline after the last item. Also, gridline -1 is the rightmost gridline, and gridline -(N+1) is the leftmost gridline. This is 1 greater than Python's positive slicing, and 1 smaller than Python's negative slicing.

In the case of `grid-column` and `grid-row`, when inserting an item into the table, you can "slice" using `a / b` syntax to specify a start and end gridline. Or you can "index" using `a` syntax, so the browser infers `b = a+1` (the item spans one track = row or column). Which is *almost* an amazing idea.  Except when `a` is -1, then `b` is inferred to be 0, not -2. And you end up with an item placed "out of bounds" and past the last column and gridline you declared. They were *so close* to achieving perfect symmetry between positive and negative indexing. At least in CSS you won't get any buffer overflows 😉

### Text field cursor affinity

A related issue is in text editing. If you're in a long paragraph and press the End key on a keyboard, the cursor will be placed after the last word in the current line, and after the space too. If you go to the next line and press the Home key, the cursor will be placed before the first word. But these 2 locations represent the same byte index into the text document! At this point, if you press the left and right arrow keys, you'll get unusual cursor behavior which differs between programs:

- Sublime Text snaps the cursor to the previous line (which I don't like).
- VS Code keeps the cursor on the current line.
- Chrome, Notepad, and Qt apps snap the cursor to the next line.
- Firefox treats "end of the current line" and "beginning of the second line" as separate locations. If you're at the end of a line (the same spot as the beginning of the next word), you need to press Right twice to get 1 character into the next word!

The same behavior occurs if a single very long word is wrapped across multiple lines. And each program listed above behaves identically, regardless if you're wrapping a paragraph or single word.

This behavior was briefly described in <https://lord.io/blog/2019/text-editing-hates-you-too/> "Affinity". That site only mentions single long words wrapped across multiple lines.

Reality is awful. There is no perfect solution.

### Ring buffers

(Note that I am not an expert on ring buffers.)

A ring buffer contains a length-N array, and (one design choice is) two pointers/indices into the array. The assumption is that the "write pointer" points to (the gridline before) the first element not written yet, and the "read pointer" points to (the gridline before) the first element which can be read. If a ring buffer has begin_ptr == end_ptr, is it empty or full? You can't tell! One solution is to always leave 1 element unwritten at all times. Another is to keep one pointer and a length counter (which ranges from 0 through N inclusive).

## Prior art

<https://wiki.c2.com/?FencePostError>

<https://news.ycombinator.com/item?id=6601515>, first comment <https://news.ycombinator.com/item?id=6602497>

---
layout: post
title: Describing convolution using item-based indexing and inclusive ranges
date: 2020-05-08 13:54 -0700
---

*This is a follow-up to my previous post, ["The gridline mental model of indexing and slicing"](../the-gridline-mental-model-of-indexing-and-slicing). I split this out because it's related to DSP as well as programming, and may not be as interesting to the broader programming audience.*

----

In some cases, it's useful to think of array indices as pointing to individual items (not fenceposts), and represent sets of items using inclusive ranges. For example, in DSP (digital signal processing), "signals" are effectively arrays of samples (AKA amplitudes) at signed-integer indices. If you have a signal of length N starting at index 0, you can treat it as an infinite signal that's only nonzero at indices `[0, N-1]` inclusive. All other indices (negative indices, and indices N and above) have a value of zero.

Convolution is a process where you "spread out" each nonzero element in a signal by an "impulse response". One example of convolution is taking a picture with a shaky or defocused camera, where we assume all objects in the image are distorted or defocused equally.

Every point of light is smudged into a blob or streak. If you assume the point of light starts at an "original position", the blob or streak is an image (two-dimensional signal) which maps positions (relative to the point of light) onto intensities. This signal is known as an "impulse response". Every object gets "smudged" by that impulse response (blob or streak). This process of "smudging" is convolving the image by the impulse response.

Convolution also applies to 1-dimensional signals like audio. Filtering or adding reverb to audio is convolving the signal by an impulse response (which is the result of sending a short impulse or pop through the filter/reverb).

If you convolve (or smudge) a one-dimensional signal of length `L` (which can only be nonzero at indices `[0, L-1]`) by an impulse response of length `P` (which can only be nonzero at indices `[0, P-1]`), the resulting signal can only be nonzero at indices `[0, (L-1) + (P-1)]` or `[0, L+P-2]`, and will have length `L+P-1`. (I think this formula also generalizes to two or more dimensions!)

## Inclusive ranges in block convolution (very technical)

Convolving a long signal by a short kernel (aka impulse response) of length `P` is often faster if you split the long signal into chunks, then compute the output in blocks of length `L`. This is because the FFT has runtime O(N log N), and one large FFT can be slower than many smaller FFTs.

One method is overlap-add convolution. If you break a long signal into blocks of length `L`, each block is only nonzero between `[0, L-1]`. And if your filter kernel has length `P` and starts at index 0, it's only nonzero between `[0, P-1]`. And if you convolve signals of length `L` and `P`, the largest index with nonzero amplitude is `L+P-2`. And the resulting signal has support `[0, L+P-2]` and length `L+P-1`.

Another example is overlap-save convolution. If you pick a block of `L+P-1` samples from the input signal, the *beginning* of each convolution output is corrupted and must be discarded. In particular, trailing samples at indices ≤ `-1` are spread out by a filter kernel of support `[0, P-1]`, corrupting outputs `[0, P-2]`, forcing you to discard the first `P-1` samples. If you don't prepend zeros to the input signal, overlap-save convolution loses the first `P-1` samples of the input/output.

## Credits

Thanks to ax6 for helping me edit this article.

---
layout: post
title: Implementing low-latency shared/exclusive mode audio output/duplex
date: 2022-06-14 00:00 -0700
---

Audio output and duplex is actually quite tricky, and even libraries like RtAudio get it wrong. If you're writing an app that needs low-latency audio without glitches, the proper implementation architecture differs between apps talking to pull-mode (well-designed, low-latency) mixing daemons, and apps talking to hardware. (I hear push-mode mixing daemons are incompatible with low latency; I discuss this at the end.) This is my best understanding of the problem right now.

## Prior art

There are some previous resources on implementing ALSA duplex, but I find them to be unclear and/or incomplete:

- <https://git.alsa-project.org/?p=alsa-lib.git;a=blob;f=test/latency.c>; gets the "write silence" part right but doesn't explain what it's doing, and the main loop is confusing.
- <https://web.archive.org/web/20211003144458/http://www.saunalahti.fi/~s7l/blog/2005/08/21/Full%20Duplex%20ALSA> gets the "write silence" part right, but doesn't know *why* it's necessary.
- <http://equalarea.com/paul/alsa-audio.html#duplexex> says: "The the interrupt-driven example represents a fundamentally better design for many situations. It is, however, rather complex to extend to full duplex. This is why I suggest you forget about all of this... In a word: JACK." However this doesn't answer the question of how _JACK_ itself implements full duplex audio.

## ALSA terminology

These are some background terms which are helpful to understand before writing an audio backend.

**Sample:** one amplitude in a discrete-time signal, or the time interval between an ADC generating or DAC playing adjacent samples.

**Frame:** one sample of time, or one sample across all audio channels.

**Period:** Every time the hardware record/play point advances by this many frames, the app is woken up to read or generate audio. In most ALSA apps, the hardware period determines the chunks of audio read, generated, or written.

However you can read and write arbitrary chunks of audio anyway, and query the exact point where the hardware is writing or playing audio at any time, even between periods. For example, PulseAudio and PipeWire's ALSA backends ignore/disable periods altogether, and instead fetch and play audio based off a variable-interval OS timer loosely synchronized with the hardware's write and play points.

- PipeWire (timer-based scheduling) experiences extra latency with batch devices ([link](https://gitlab.freedesktop.org/pipewire/pipewire/-/wikis/FAQ#pipewire-buffering-explained)), and PulseAudio used to turn off timer-based scheduling for batch devices ([link](https://www.alsa-project.org/pipermail/alsa-devel/2014-March/073816.html)).
- On the other hand, Paul Davis says conventional *period-based* scheduling struggles *more* than timer-based (PulseAudio, PipeWire) for batch devices ([link](https://blog.linuxplumbersconf.org/2009/slides/Paul-Davis-lpc2009.pdf) @ "The Importance of Timing"). I'm not sure how to reconcile this.

**Batch device:** Represented by `SNDRV_PCM_INFO_BATCH` in the Linux kernel. I'm not exactly sure what it means. <https://www.alsa-project.org/pipermail/alsa-devel/2014-March/073816.html> says it's a device where audio can only be sent to the device in period-sized chunks. <https://www.alsa-project.org/pipermail/alsa-devel/2015-June/094037.html> is too complicated for me to understand.

**Quantum:** PipeWire's app-facing equivalent to ALSA/JACK periods.

**Buffer size:** the total amount of audio which an input ALSA device can buffer for an app to read, or can be buffered by an app for an output ALSA device to play. Always at least 2 periods long.

**Available frames:** The number of frames (channel-independent samples) of audio readable/buffered (for input streams) or writable (for output streams).

**"Buffered" frames:** For input devices, this matches available (readable) frames. For output devices, this equals the buffer size minus available (writable) frames.

**hw devices, plugins, etc:** See <https://www.volkerschatz.com/noise/alsa.html>.

## Minimum achievable input/output/duplex latency

The minimum achievable audio latency at a given period size is achieved by having 2 periods of total capture/playback buffering between hardware and a app (RtApiAlsa, JACK2, or PipeWire).

- If an audio daemon mixes audio from multiple apps, it can only avoid adding latency if there is no buffering (but instead synchronous execution) between the daemon and apps. JACK2 in synchronous mode and PipeWire support this, but pipewire-alsa fails this test by default, so ALSA is not a zero-latency way of talking to PipeWire.

For duplex streams, the total round-trip (microphone-to-speaker) latency of a duplex stream is `N` periods (the maximum amount of buffered audio in the output buffer). `N` is always ≥ 2 and almost always an integer.

For capture and duplex streams, there are `0` to `1` periods of capture (microphone-to-screen) latency (since microphone input can occur at any time, but is always processed at period boundaries).

For playback and duplex streams, there are `N-1` to `N` periods of playback (keyboard-to-speaker) latency (since keyboard input can occur at any point, but is always converted into audio at period boundaries).

These values only include delay caused by audio buffers, and exclude extra latency in the input stack, display stack, sound drivers, resamplers, or ADC/DAC.

Note that this article doesn't cover the advantages of extra buffering, like smoothing over hitches, or JACK2 async mode ensuring that an app that stalls won't cause the system audio and all apps to xrun. I have not studied JACK2 async mode though.

## Avoid blocking writes (both exclusive and shared, output only)

If your app generates one output period of audio at a time and you want to minimize keypress-to-audio latency, regardless if your app outputs to hardware devices or pull-mode daemons, it should never rely on blocking writes to act as output backpressure. Instead it should wait until 1 period of audio is writable, *then* generate 1 period of audio and nonblocking-write it. (This does not apply to duplex apps, since waiting for available _input_ data effectively acts as *output* throttling.)

If your app generates audio *before* performing blocking writes for throttling, you will generate a new period of audio as soon as the previous period of audio is written (a full period of real time before a new period of audio is writable). This audio gets buffered for an extra period (while `snd_pcm_writei()` blocks) before reaching the speakers, so **external (eg. keyboard) input takes a period longer to be audible.**

(Note that avoiding blocking writes isn't necessarily beneficial if you don't generate and play audio in chunks synchronized with output periods.)

**Issue:** RtAudio relies on blocking `snd_pcm_writei` in pure-output streams. This adds 1 period of keyboard-to-speaker latency to output streams. (It also relies on blocking `snd_pcm_writei` for duplex streams, but this is essentially harmless since RtAudio first blocks on `snd_pcm_readi`, and by the time the function returns, if the input and output streams are synchronized `snd_pcm_writei` is effectively a nonblocking write call.)

### ALSA: blocking reads/writes vs. snd_pcm_wait() vs. poll()

Making a blocking call to `snd_pcm_readi()` before generating sound is basically fine and does not add latency relative to nonblocking reads (`snd_pcm_sw_params_set_avail_min(1 period)` during setup, and calling `snd_pcm_wait()` before every read).

On the other hand, generating sound then making a blocking call to `snd_pcm_writei()` (in output-only streams) adds a full period of keyboard-to-speaker latency relative to nonblocking writes (`snd_pcm_sw_params_set_avail_min(unused_buffer_size + 1 period)` during setup, and calling `snd_pcm_wait()` before generating and writing audio).

`poll()` has the same latency as `snd_pcm_wait()` and is more difficult to setup. The advantage is that you can pass in an extra file descriptor, allowing the main thread to interrupt the audio thread if `poll/snd_pcm_wait()` is stuck waiting on a stalled ALSA device. (I'm not sure if stalled ALSA is common, but I've seen stalled shared-mode WASAPI happen.)

## Avoid buffering shared output streams (output and duplex)

Most apps use shared-mode streams, since exclusive-mode streams take up an entire audio device, preventing other apps from playing sound. Shared-mode streams generally communicate with a userspace audio daemon[^1], which is responsible for mixing audio from various programs and feeding it into hardware sound buffers, and ideally even routing audio from app to app.

[^1]: ALSA dmix may be kernel-based. I'm not sure, and I haven't looked into it.

If an app needs a output-only or duplex shared-mode stream, and must avoid unnecessary output latency, it should not buffer output audio itself (or generate audio *before* performing a blocking write, discussed above). Instead it should wait for the daemon to request output audio (and optionally provide input audio), *then* generate output audio and send it to the daemon. This minimizes output latency, and in the case of duplex streams, enables *zero-latency* app chaining between apps in an audio graph! To achieve this, the pull-mode mixing daemon (for example JACK2 or PipeWire) requests audio from the first app, and synchronously passes it to later apps within the *same period* of real-world time. Sending audio through two apps in series has zero added latency compared to sending audio through one app. The downside is that if you chain too many apps, JACK2 can't finish ticking all the apps in a single period, and fails to output audio to the speakers in time, resulting in an audio glitch or xrun.

**Issue:** Any ALSA app talking to pulseaudio-alsa or pipewire-alsa (and possibly any PulseAudio app talking to pipewire-pulse) will perform extra buffering. Hopefully RtAudio, PortAudio, etc. will all add PipeWire backends someday (SDL2 already has it: <https://www.phoronix.com/scan.php?page=news_item&px=SDL2-Lands-PipeWire-Audio>).

As a result, for the remainder of the article, I will be focusing on using ALSA to talk to *hardware* devices.

## Buffer 1-2 periods in exclusive output streams (output and duplex)

It is useful for some apps to open hardware devices directly (such that no other app can output or even receive audio), using exclusive-mode APIs like ALSA. These apps include audio daemons like PipeWire and JACK2 (which mix audio output from multiple shared-mode apps), or DAWs (which occupy an entire audio device for low-latency low-overhead audio recording and playback).

Apps which open hardware in exclusive mode must handle output timing in real-world time themselves. They must read input audio as the hardware writes it into buffers, and send output audio to the buffers *ahead* of the hardware playing it back.

In well-designed duplex apps that talk to hardware, such as jack2 talking to ALSA, the general approach is:

- Pick a mic-to-speaker delay (called `used_buffer_size` and measured in frames).
- Pick a period size, which divides `used_buffer_size` into `N` periods. `N` is usually an integer ≥ 2.
- Tell ALSA to allocate an input and output buffer, each of size ≥ `used_buffer_size`, each with the correct period size.
- Write `used_buffer_size` frames of silence to the output

Then loop:

- wait for 1 period/block of input to be available/readable, and 1 period/block of output to play and be available/writable. JACK2 uses `poll()`, if you don't need cancellation you can use `snd_pcm_wait()` or even blocking `snd_pcm_readi()`.
- read 1 period of input, and pass it to the user callback which generates 1 period of output
- write 1 period of output into the available/writable room

## Implementing exclusive-mode duplex like JACK2

JACK2's ALSA backend, and this guide, assume the input and output device in a duplex pair share the same underlying sample clock and never go out of sync. Calling `snd_pcm_link()` on two streams is supposed to succeed if and only if they share the same sample clock, buffer size and period count, etc. (the exact criteria are undocumented, and I didn't read the kernel source yet). If it succeeds, it not only starts and stops the streams together, but is supposed to synchronize the input's write pointer and the output's read pointer.

PipeWire supports rate-matching resampling ([link](https://gitlab.freedesktop.org/pipewire/pipewire/-/wikis/FAQ#how-are-multiple-devices-handled)), but (like timer-based scheduling) it introduces a great deal of complexity (*heuristic* clock skew estimation, resampling latency compensation), which I have not studied, is out of scope for opening a simple duplex stream, and *actively detracts* from learning the fundamentals.

Note that `unused_buffer_size > 0` is also incidental complexity, and not essential to understanding the concepts. Normally `buffer_size = N periods`.

On ALSA, you can implement full duplex period-based audio by:

- Optionally(?) open input and output `snd_pcm_t` in `SND_PCM_NONBLOCK`.
- Setup both the input and output streams with `N` periods of audio. `N` is selected by the user, and is usually 2-4. (If the device only supports `>N` periods of audio, JACK2 can open the device with `>N` periods, but simulate `N` periods of latency by never filling the output device beyond `N` periods.)
- Let `used_buffer_size = N periods` (in frames). This equals the total `buffer_size` unless the device only supports `>N` periods.
- Let `unused_buffer_size = buffer_size - used_buffer_size` (in frames). This equals 0 unless the device only supports `>N` periods.
- Set up the input and output streams, so software waiting/polling will wake up when the hardware writes or reads the correct amount of data.
	- For the input stream, we want to read as soon as 1 period of data is readable/available, so call `snd_pcm_sw_params_set_avail_min(1 period)`. You can skip this call if you open the device without `SND_PCM_NONBLOCK` and use blocking `snd_pcm_readi`, but to my knowledge `snd_pcm_sw_params_set_avail_min()` is not optional in the lower-overhead mmap mode.
- The output stream is more complicated if `unused_buffer_size != 0`.
	- We want to write 1 period of audio once `buffered ≤ used_buffer_size - 1 period` (in frames). And we know `writable/available = buffer_size - buffered`. So we want to write audio once `writable/available ≥ unused_buffer_size + 1 period`.
	- Call `snd_pcm_sw_params_set_avail_min(unused_buffer_size + 1 period)`, so polling/waiting on the output stream will unblock once that much audio is writable.
- For duplex streams, write `N` periods of silence. This can be skipped for output-only streams, but JACK2 does it for those too.
- `snd_pcm_start()` the input stream if available, and the output stream if available and not linked to the input.

And in the audio loop:

- Either call `poll()` (like JACK2, can wait on multiple fds) or `snd_pcm_wait` (simpler, synchronous), to wait until 1 period of room is readable from the input stream and writable to the output stream (excluding `unused_buffer_size`).
	- At this point, we have `N-1` periods of time to generate audio, before the input buffer runs out of room for capturing audio and the output runs out of buffered audio to play. This is why `N` must be greater than 1; if not we have *no* time to generate 1 period of audio to play.
- Read 1 period of audio from the input buffer, generate 1 period of output audio, and write it to the output buffer.
	- Now the output buffer holds `≤ used_buffer_size` frames, leaving `≥ unused_buffer_size` room writable/available.

### RtAudio gets duplex wrong, can have xruns and glitches

**Issue:** RtAudio opens and polls an ALSA duplex stream (in this case, duplex.cpp with [extra debug prints added](https://github.com/nyanpasu64/rtaudio/tree/alsa-duplex-buffering), opening my motherboard's hw device) by:

- Don't fill the output with silence.
- Call `snd_pcm_sw_params_set_start_threshold()` on both streams (though RtAudio only triggers on the input, which starts both streams).
- `snd_pcm_link()` the input and output streams so they both start at the same time. Setup the streams the same way regardless if it succeeds or fails. (On my motherboard audio, it succeeds.)

Then loop:

- Call `snd_pcm_readi(1 period)` of input (blocking until available), and pass it to the user callback which generates 1 period of output.
    - Because RtAudio calls `snd_pcm_sw_params_set_start_threshold` on the input stream, and the two streams are linked, `snd_pcm_readi()` starts both the input and output streams *immediately* (upon call, not upon return). The output stream is started with no data inside, and tries to play the absence of data. It's a miracle it doesn't xrun immediately.
    - Once the input stream has 1 period of input, `snd_pcm_readi` returns. By this point, the output stream has more `snd_pcm_avail()` than the total buffer size, and *negative* `snd_pcm_delay()`, yet *somehow* it does not xrun on the first `snd_pcm_writei()`.
- Call `snd_pcm_writei(1 period)` of output. This does not block since there are three periods available/writable (or two if the input/output streams are not linked).
    - This is supposed to be called when there is 1 period of empty/available space in the buffer to write to. Instead it's called when there is 1 period of empty space *more* than the entire buffer size! I don't understand how ALSA even allows this.

### Fixing RtAudio output and duplex

To resolve this for duplex streams, the easiest approach is to change stream starting:

- Write 1 full buffer (or the used portion) of silence into the output.
- Don't call `snd_pcm_sw_params_set_start_threshold()` on the output stream of a duplex pair. Instead use `snd_pcm_link()` to start the output stream upon the first input read (or if `snd_pcm_link()` fails, start the output stream yourself before the first input read).

This approach fails for output-only streams. To resolve the issue in both duplex and output streams, you must:

- Call `snd_pcm_sw_params_set_avail_min(unused_buffer_size + 1 period)` before starting the output stream.
- Call `snd_pcm_wait()` (or `poll()`) on the output stream every period, *before* generating audio.

I haven't looked into how RtAudio stops ALSA streams (with or without `snd_pcm_link()`), then starts them again, and what happens if you call them quickly enough that the buffers haven't fully drained yet.

## (optional) Replacing blocking reads/writes with cancellable polling

RtAudio needs to use polling to avoid extra latency in output-only streams. Should it be used for duplex and input-only streams as well? Is it worth adding an extra pollfd for cancelling blocking writes (possibly replacing the condvar)?

I don't know how to refactor RtAudio to allow cancelling a blocked `snd_pcm_readi/writei`. Maybe pthread cancellation is sufficient, I don't know. If not, one JACK2 and cpal-inspired approach is:

- Open all `snd_pcm_t` in `SND_PCM_NONBLOCK`
- Fetch fds for each `snd_pcm_t` using `snd_pcm_poll_descriptors()`
- Share an interrupt pipefd/eventfd between the GUI and audio thread
- In the audio callback:
	- `poll()` the input, output, and interrupt fds
	- Pass the result into `snd_pcm_poll_descriptors_revents()`
	- Only perform non-blocking PCM reads/writes, or exit the loop if the interrupt fd is signalled.

Unfortunately this requires a pile of refactoring for relatively little gain.

## Is RtAudio's current approach appropriate for low-latency pipewire-alsa?

**Update: No.**

pipewire-alsa in its current form ([774ade146](https://gitlab.freedesktop.org/pipewire/pipewire/-/commit/774ade1467b8c68ac9646624d941be994bd3702b)) is wholly unsuitable for low-latency audio.

I use `jack_iodelay` to measure signal latency, by using Helvum (a PipeWire graph editor) to route `jack_iodelay`'s output (which generates audio) through other nodes (which should pass-through audio with a delay) and back into its input (which measures audio and determines latency). When `jack_iodelay` is routed through hardware alone, it reports the usual 2 periods/quantums of latency. When I start RtAudio's ALSA duplex app with period matched to the PipeWire quantum (which should add only 1 period of latency since `snd_pcm_link()` fails), and route `jack_iodelay` through hardware and duplex in series, `jack_iodelay` reports a whopping 7 periods of latency. My guess is that pipewire-alsa adds a full 2 periods of buffering to both its input and output streams. I'm not sure if I have the motivation to understand and fix it.

**Earlier:**

RtAudio doesn't write silence to the output of a duplex stream before starting the streams, and only writes to the output stream once one period of data arrives at the input stream. This is unambiguously wrong for hw device streams. Is it the best way to achieve zero-latency alsa passthrough, when using the pipewire-alsa ALSA plugin? I don't know if it works or if the output stream xruns, I don't know if this is contractually guaranteed to work, and I'd have to test it and read the pipewire-alsa source ([link](https://gitlab.freedesktop.org/pipewire/pipewire/-/blob/master/pipewire-alsa/alsa-plugins/pcm_pipewire.c)).

Is it possible to achieve low-latency *output-only* ALSA, perhaps by waiting until the buffer is entirely empty (`snd_pcm_sw_params_set_avail_min()`)? Again I don't know, and I'd have to test.

## Push-mode audio loses the battle before it's even fought

I hear push-mode mixing daemons like PulseAudio (or possibly WASAPI) are fundamentally bad designs, incompatible with low-latency or consistent-latency audio output.

<https://superpowered.com/androidaudiopathlatency> ([discussion](https://news.ycombinator.com/item?id=9386994)) is an horror story. In fact I read elsewhere that pre-AAudio Android duplex loopback latency is *different* on every run; I can no longer recall the source, but it's entirely consistent with the user application's own ring buffering, or if input and output streams were started separately and not started and run in sync at a driver level like `snd_pcm_link`.

Note that Android audio may have improved since then, see AAudio and <https://android-developers.googleblog.com/2021/03/an-update-on-androids-audio-latency.html>.

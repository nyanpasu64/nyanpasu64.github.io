---
title: 'ExoTracker Newsletter #2 - Pivoting to SNES, designing an instrument list'
---

For those of you who aren't already aware, ExoTracker is a tracker-like composing tool, based around subdividing beats instead of integer rows. This allows the user to place notes at arbitrary fractions of a beat (like sheet music), and additionally allows tracker-like delay effects (which can be negative, which is impossible in most trackers). Beat subdivision allows for mixing eighth notes and triplets, and using beats for timing (rather than rows) could make tempo calculation more intuitive than other trackers.

## Pivoting to a SNES tracker

After spending several months away from ExoTracker, I've decided to switch away from emulating a Famicom with expansions, to a SNES's SPC700 sound chip. I chose to do this because the SNES has less pre-existing options for composing (especially if you limit yourself to free options, ruling out chipsynth SFC and somewhat SNES Tracker). Another benefit is that it's simpler to write a SNES sound engine; the SNES only has 1 type of channel, so I don't need to find a way to modularize/abstract the sound driver to reuse instrument code for the Famicom's numerous expansion chips, which have different register addresses, sizes, and interpretations (pitch: period vs frequency vs Yamaha, volume: linear vs. log vs. hardware envelopes).

Issue is, I haven't decided how to handle timing... On the NES, the vblank interrupt is the most processor-efficient way to tick the audio engine, and you normally run one tick per vblank. But (to the best of my understanding) the S-SMP (CPU) has 1 fast and 2 slow timers (with configurable dividers), and they don't interrupt the S-SMP, so you need to busy-wait and poll them manually. And some SNES games change the timer speed to adjust song tempo (so each quarter note is a fixed number of timer ticks like MIDI). Others have unchanging timer speeds and let an uneven number of timer ticks pass between each subsequent quarter note (like FamiTracker's tempo).

Looking at how pre-existing trackers behave, FamiTracker allows users to configure Speed and Tempo, which interact strangely[^1]. 0CC-FT adds more modes: "fixed" to turn off Tempo so Speed controls "ticks/row" directly, and grooves to switch Speed on every row.

OpenMPT has [3 tempo modes](https://wiki.openmpt.org/Manual:_Song_Properties#Overview) comprising 2 conceptually different types: Classic/Alternative let users pick the duration of rows, whereas Modern lets users pick the duration of beats. Both come with customizable "ticks/second" (which is fixed in FamiTracker), and Classic/Alternative (but not Modern) suffer from tempo rounding errors. Since ExoTracker doesn't have rows (but instead arbitrary beat subdivisions), copying Classic/Alternative is not an option, and only copying Modern is.

I'm going to use a SPC700 emulation core (likely Blargg's). I think only the S-SMP can read the SPC700's timers... but I probably won't write S-SMP code, but instead will reimplement the driver in C++, using native x86 instructions to communicate with the GUI and S-DSP emulator, so I'll have to simulate the timers myself. Anyway I need to pick whether to use the fast or slow timer (probably copy existing games), what GUI to provide for customizing the timer rate (either expose the raw register value, or a tempo which gets converted/rounded to a divider register),  and what tempo mode to use (fixed-timer/FamiTracker tempo, vs variable-timer/MIDI/OpenMPT Modern) since it's impractical to implement multiple tempo modes in the C++ and ASM drivers.

[^1]: In FamiTracker:

    - Tempo only matches "beats/min" if Speed * "Highlight 1" = 24.
    - Speed only matches "ticks/row" if Tempo = "ticks/second" * 2.5.
    - Speed defaults to 6 "ticks/row". Highlight 1 defaults to 4 rows/beat. Tempo defaults to 150 "beats/min". Ticks/second defaults to 60 (or 50 on PAL) because ticks are usually triggered by vblanks/frames.

## Designing an instrument list

There is no instrument editor, and I don't know when there will be one. In the meantime I've been working on adding an instrument list.

{% include img-125.html src="instrument-list.png" alt="I added an instrument list widget. The instruments form columns, just like FamiTracker's instrument list. Unlike FamiTracker, it shows all 128 instrument slots, even though most of them are empty. It's confusing to look at." %}

So looking at this picture, obviously it needs improvement. Aside from showing dozens of empty slots, another difference from FamiTracker is that each column has its own width, instead of matching the width of the widest instrument in any column. I'm not sure if that's a good or bad thing, or if the Qt GUI library allows me to change it.

One solution is to copy how FamiTracker only shows occupied instrument slots. Implementing will take work, because you need to filter the array of numbered instruments and only expose the slots with instruments, and when a user clicks an item, map from the item's position in the widget back to instrument numbers. One approach is to keep a cached vector of items, each one holding an instrument number that's guaranteed to point to a non-empty instrument, and regenerate this vector whenever the document is modified.

### Missing functionality in FamiTracker

Unfortunately FamiTracker's instrument drag-and-drop behavior leaves features to be desired. FamiTracker defines drag-and-drop to swap instruments (and not empty slots). But sometimes I want to move an instrument into an empty slot, which is not possible (unless you fill empty slots with placeholder instruments). And sometimes I want to insert, remove, or move instruments, which shifts all instrument numbers afterwards by 1. (In some cases, this may even include empty slots as well, which may or may not be desirable.) This is not possible in FamiTracker unless you drag each instrument over one by one, which is tedious.

{% include img-100.html src="famitracker-instrument-groups.png" alt="At 96 DPI, FamiTracker's instruments are grouped into 8-instrument columns. Each has the same leading digit, and each leading digit is split into exactly 2 columns." %}

I also like to categorize instruments into percussion, melodic, and expansion chip instruments, then divide them into groups of 8. This is because in FamiTracker, the instrument list is rendered as groups of 8 instruments. However since FamiTracker does not render empty instrument slots, this requires creating empty instruments to fill in any gaps in the numbering scheme.
<!-- I also use empty instruments in FamiTracker to group instruments. In FamiTracker, there's 8 instruments in each column, so i can organize them in groups of 8. However this requires creating empty instruments in between. -->
<!-- Another benefit of placeholder instruments (or showing empty slots) is for grouping instruments.  -->

(Sidenote: If the list widget isn't exactly 8 instruments tall, this grouping system breaks, and the instruments are no longer arranged in visually neat columns corresponding 0x0 through 0x7 and 0x8 through 0xf.)

{% include img-125.html src="famitracker-instrument-groups-125.png" alt="Unfortunately at 120 DPI, FamiTracker's instrument list is 9 instruments tall rather than 8 (due to rounding differences), breaking the groups. At 192 DPI, the list is 10 instruments tall!" %}

### Solution: showing placeholders?

One possibility is providing a user option to show all instruments, including placeholders, from zero until the last occupied instrument slot. You can create or delete instruments in-place (filling or creating an empty slot), drag-and-drop to swap slots (both empty and full), and even insert, delete, or move instruments while shifting the rest forwards or backwards.

If you want to create or insert instruments past the largest-numbered slot, you'll have to check a box to show all slots, even unoccupied ones. This will look ugly if empty columns are very narrow, but will look less ugly if all columns are the same width.

Unfortunately this solution doesn't have the same properties as the "empty-named instruments" I've been using in FamiTracker, requiring users to adjust. If you're using empty-but-shown instrument slots (instead of empty-named instruments as in FamiTracker), then pressing the "New Instrument" button won't append an instrument to the end of the list (after all the empty-name instruments), but will instead fill the first empty slot.

### Another approach: OpenMPT

OpenMPT has a tree view on the left of the window, showing a list of numbered samples, and (in many module formats) a list of numbered instruments. The numbers are integers starting from 1, unlike FamiTracker's hex values starting from 00. OpenMPT behaves like a dynamic-size list of samples/instruments which may have empty names and no data. Contrast this with my previous idea of a fixed-size list of instruments, where each may be absent.

I've run some testing in a .mptm file on OpenMPT 1.29.07. It seems simple at first, but gets weirder the further you investigate.

- Right-clicking any sample/instrument and clicking "Insert Sample" or "Insert Instrument" will insert one *after* the one you've clicked, increasing the number of each subsequent sample/instrument by 1.
    - This makes sense under my proposed instrument scheme, and is not possible in FamiTracker.
- Inserting a sample/instrument at the very end of the list will create a blank sample/instrument (with a dimmed icon) at the end of the list. This can be repeated to add multiple blank samples/instruments.
    - This shows that OpenMPT displays (and probably stores) the sample/instrument lists with a variable "length" field. This functions quite differently than my proposed "show trailing placeholders" checkbox, and I suspect OpenMPT's UI is better. OpenMPT makes it easier to append instruments, whereas my code makes it easier to insert instruments at large indices without filling the space before it.
    - I'm concerned that copying OpenMPT's approach can lead to bugs. If my code stores instruments in a dynamic-length vector, it's easy to index out of bounds (which can be avoided if I use custom getter functions that treat out-of-bounds indices as "no instrument present"). If my code stores instruments in a fixed-size array with a cosmetic length field (whether saved in the module or not), I can accidentally set a length shorter than the index of the largest instrument present + 1.
- Right-clicking any sample/instrument and clicking "Delete Sample/Instrument" replaces it with an empty slot (with a dimmed icon), and does not shift future instruments/samples back by 1 to fill the hole generated. The exception is deleting the last sample/instrument, which will decrease the list's length by 1 and not leave behind an empty slot.
    - It would be nice to have a way to delete full/empty slots and shift everything backwards by 1.

Samples have strange behavior:

- Samples with no wave data have dimmed icons. This doesn't mean much.
- Opening the Samples tab and clicking the Insert Sample button (not to be confused with the Insert Sample right-click menu item) will *sometimes* insert one at the end of the list of visible samples... and *other times* overwrite samples with no name and no sample data (created by "Insert Sample" or "Delete Sample") (and only insert a new sample if none exist). The resulting sample will be called `untitled`, and because of the non-empty name, cannot be overwritten or deleted.
    - Bizzare.
- Deleting any sample will delete all trailing "empty" samples (no name *and* no waveform). The only exception is if all samples in a module are empty, in which case it leaves one behind (all modules have at least 1 sample, and OpenMPT will never delete the last sample).

Instruments have a different set of strange behavior:

- Instruments which have just been deleted have dimmed icons. The "Insert Instrument" *button* will insert an instrument into the first dimmed instrument, and append a new one if none exist. Changing any property of a deleted/dimmed instrument (name, contents) will undim its icon permanently (until deleted again). The "Insert Instrument" *menu item* will undim *all* icons.
    - So much for consistency.
- Trying to delete an instrument slot does nothing if it's dimmed.
- Deleting any instrument does not clear trailing empty instruments. Deleting the last instrument shrinks the list by 1 instead of dimming the last instrument, but if the second-last instrument was dimmed, it turns into the last instrument and remain dimmed. In this scenario, you cannot shrink the list any further; the Delete key does nothing, and the right-click menu does nothing. The only solution is to undim the icons and then delete them.

### Sidenote: OpenMPT bugs

In the sidebar, click a sample. Click again (opens a rename field with time delay) and rapidly press Delete before the rename is initiated. The rename will pop up after the delete dialog appears. Clicking Yes to delete will delete the instrument, but keep the rename field open.

I've gotten OpenMPT to omit a number in the sidebar's sample/instrument numbering scheme (probably samples, forgot), after messing with it. It reappeared when I pressed F5 (which began playback).

## Footnotes

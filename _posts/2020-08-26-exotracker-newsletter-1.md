---
layout: post
title: 'ExoTracker Newsletter #1'
date: 2020-08-26 01:46 -0700
---

I just finished implementing timeline entry editing. Since I have school coming up, I decided to release a demo of its current state. Since my summary was getting a bit too long to post in Discord, I decided to write a blog post / newsletter.

## Demo download

Windows 64-bit: <https://ci.appveyor.com/api/buildjobs/bb37g6b38wh88m9f/artifacts/exotracker-v1.0.58-dev.7z>

Source: <https://gitlab.com/nyanpasu64/exotracker-cpp/-/tree/timeline-editor> (currently commit dd0f7377) It only compiles in recent GCC and Clang (only tested Clang 10), due to using statement expressions.

## Demo notes

- Only Famicom/NES APU1 is supported. Some demo songs have dual APU1 which can be used for composing.
- Try passing in names of sample documents as command-line arguments. Listed in order from most to least useful:
    - Partial songs: `dream-fragments`, `world-revolution` (default song)
    - `empty` (add your own notes)
    - `audio-test` (dual APU1) (sounds bad, but useful for finding audio stuttering)
    - `block-test` (dual APU1) (rendering test for block system, no notes or events)
    - `render-test` (sounds bad, negative octave text is too wide for the screen)
- Some sample documents have short and/or looped blocks (the gray rectangles to the left of each channel), which are not possible in most other trackers (I don't know if LSDj and C64 trackers support this). But right now, users can only create full-grid blocks, and cannot delete blocks.
    - The block system is powerful, but unfortunately not editable through the UI yet, so you can't try it out to see useful it is.
    - Pattern reuse is not implemented.
- All edits are undoable. Some but not all timeline edits save cursor position.
- There are buttons for reordering timeline entries. They're supposed to have icons instead of text, but icons are only available on my machine.
- The actual timeline widget (list of rows) is unfinished and will be replaced with a custom-drawn widget.
- The audio code will lock up if you decrease the timeline row length until a block has a negative length 😉

## Timeline system overview

**tl;dr skip forward to "Demo feedback" if you want to just play with the program instead of reading documentation.**

The frame/order editor is replaced with a timeline editor, and its functionality is changed significantly.

The pattern grid structure from existing trackers is carried over (under the name of timeline rows and grid cells). Each timeline row has its own length which can vary between rows (like OpenMPT, unlike FamiTracker). Each timeline row holds one timeline cell (or grid cell) per channel. However, unlike patterns, timeline cells do not contain events directly, but through several layers of indirection.

A timeline cell can hold zero or more blocks, which carry a start and end time (in integer beats) and a pattern. These blocks have nonzero length, do not overlap in time, occur in increasing time order, and lie between 0 and the timeline cell's length (the last block's end time can take on a special value corresponding to "end of cell")[1].

Each block contains a single pattern, consisting of a list of events and an optional loop duration (in integer beats). The pattern starts playing when absolute time reaches the block's start time, and stops playing when absolute time reaches the block's end time. If the loop duration is set, whenever relative time (within the pattern) reaches the loop duration, playback jumps back to the pattern's begin. A block can cut off a pattern's events early when time reaches the block's end time (either the pattern's initial play or during a loop). However a block cannot start playback partway into a pattern (no plans to add support yet).

Eventually, patterns can be reused in multiple blocks at different times (and possibly different channels).

[1] I'm not sure what to do if a user shrinks a timeline row, which causes an numeric-end block to end past the cell, or an "end of cell" block to have a size ≤ 0, etc.

### Motivation

The timeline system is intended to allow treating the program like FamiStudio or a tracker, with timestamps encoded relative to current pattern/frame begin, and reuse at pattern-level granularity. If you try to enter a note/volume/effect in a region without a block in place, a block is automatically created in the current channel, filling all empty space available (up to an entire grid cell) (not implemented yet).

It is also intended to have a similar degree of flexibility as a DAW like Reaper (fine-grained block splitting and looping). The tradeoff is that because global timestamps are relative to grid cell begin, blocks are not allowed to cross grid cell boundaries (otherwise it would be painful to convert between block/pattern-relative and global timestamps).

## Unresolved questions

- Are the gray block rectangles (to the left of each channel) ugly? I'm planning to use those to allow dragging patterns around, resizing them, and distinguishing reused patterns through color.
- Should I rename the timeline to something else?
    - Sequence?
    - Order? (I feel it's bad because "order" implies every entry is merely the ID of a single pattern, but in reality is a container for 0 or more loopable patterns.)
    - OpenMPT has an "order list" widget to edit a "sequence" of patterns; it uses two names for similar concepts.
- What should I call a row/unit in the timeline editor? It's treated as a coarse unit of time and a container for blocks/patterns, but is not a pattern.
    - Grid row? Timeline/sequence row? (I keep confusing "timeline row" with "pattern row". "Grid" is concise, but I don't know if it's an unintuitive name.)
    - Segment?
    - (Timeline/sequence) entry?
    - Cell? (currently used for "one row, one channel")
- How should I improve my current behavior when adding and deleting timeline entries?
    - Should adding a new timeline entry move the cursor to the new entry's row 0? Should undoing move the cursor to the same spot, move it to the old location, or leave the cursor in place?
    - Should deleting an timeline entry move the cursor to the former entry's row 0? Should undoing move the cursor to the same spot, move it to the old location, or leave the cursor in place?
    - Eventually I'll add the ability to right-click and add/delete timeline entries other than the one the cursor is in. Should the cursor move to the right-clicked entry, and stay there after undoing?
- Once the timeline widget is implemented, what should it show?
    - Titles for each sequence entry?
    - "Pattern overview" with coarse-grained visualizations of blocks (gray for unique blocks, colored for shared)? Should shared patterns have numbers? Names? Should all patterns have numbers (an idea I'm not a fan of)?
    - Draw both (and somehow try to find enough room for both)?
    - 0CC has bookmarks and highlights, but doesn't show all names.

## Feedback

If you find any crash bugs, let me know. (Some tricky-to-get-right areas were deleting the last row in the timeline, or deleting a long row and the cursor moves into the next, shorter, row.)

If you have any UI or behavior suggestions, tell me too. (I personally think I got the code reasonably watertight, but the UI behavior is a toss-up and I have no clue how people will react.)
# MPEcreator — working handover

Started 21 October 2026. A planned Lua/ReaImGui script for REAPER that highly automates adding MPE to the notes of a MIDI item.
Status: idea phase, no code yet. Everything decided so far is in `MPEcreator_idea.md` (read it first); this file is the short form.

## Who and how

tjingboem, composer and developer, Groningen. Terse prose, no bullet lists in replies, no emojis, never the word "honest". Evidence
before code. He tests before committing and commits only when he says "commit and push". Commit messages end with
`Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`. Never kill REAPER. `gh` auth is broken; SSH git works.
Remote: `git@github.com:tjingboem/MPEcreator.git`, branch `master`.

## What it is

Not an editor. The user draws nothing note by note; he sets rules and the tool applies them to every note. The tool detects the
note-on moment, channel, pitch and duration of each note (like the red diamonds in ReaShapeM) and writes MPE: glide (pitch bend),
slide (CC74), press (channel pressure), lift (note-off velocity), and strike (note-on velocity) if wanted.

## Decided

1. At note-off the bend stays as it is. Nothing is reset at the end of a note. The next note on that channel writes its own start
   values just before its note-on (and pressure and slide again right after it).
2. Bends are per channel, so notes on one channel are first spread over channels 2 to 16 (channel 1 reserved) with a conflict graph:
   overlapping or touching notes never share a channel; seeded, so the result is repeatable.
3. Bend range: up to 96 semitones (14 bits is then 1.17 cents per step). Amount limits are set in semitones and converted with the range.
4. Rules are masks, as in ReaMaskM: each rule is a pair of Min and Max envelopes over the item, the range between them can change over
   time. Min above Max collapses to Max (not swapped). A value outside a hard bound is skipped, not clamped. Numbers are shown while
   dragging. Presets in the style of `.mask`; an audition runs the real writer.
5. Glide rules so far: where in the note (a range), how many bends (1 to 5), direction as two probability envelopes (chance of up, chance
   of down) over the item. How the two chances combine is OPEN (one roll per note, two independent rolls, or one roll per bend;
   Claude recommends one roll per bend).
6. The note reader of ReaShapeM becomes a shared library for ReaShapeM and MPEcreator. Not yet looked at how to lift it out.

## Learned from MPElodyne (someone else's MPE editor, licence unknown)

Ideas only; do not copy its code without asking the author. Read and write the raw event stream (`MIDI_GetAllEvts` / `MIDI_SetAllEvts`),
re-read the own output and refuse to write if ownership is ambiguous, event priorities at the same tick, pitch bend as semitones scaled
by the range, channel or polyphonic pressure, microtuning offsets as a base bend, a modulation stack with integrated rate, a pure-Lua
core tested with `lua5.4`.

## Reuse candidates (not yet checked for how easily they lift out)

ReaMaskM: envelope canvas, Min/Max collapse, `.mask` presets, AUDITION, MicroTuning with its MPE channel allocator. ReaNvelopes: curve
engine (`Curve.lua`), DRAW, OSC, RAND, `CurveOut.commitLift` for the note-off velocity. ReaSnap: Integration Contract (pieces 1 to 5).

## Open

The combining rule for up and down. Which bend range ReaSamp's MPE Tabs use in practice (the tool assumes up to 96). The licence of
MPElodyne. Whether the DRAW code is shared as a library or copied. A build order is at the end of `MPEcreator_idea.md`.

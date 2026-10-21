# MPEcreator — idea, 21 October 2026

Status: an idea only, not designed in detail, not started. Written after tjingboem asked whether ReaNvelopes has
everything to create MPE data. It does not, and he asked for the answer to be kept as a separate idea, a tool of its own
(working name MPEcreator), not as a feature of ReaNvelopes.

## What it would do

It takes the notes of a MIDI item and turns them into MPE: every sounding note gets its own channel, and every note gets
its own pitch bend, pressure and slide (CC74) gesture on that channel, for exactly the length of the note. The result is
an item that plays as an MPE performance on any MPE instrument, ReaSamp first (Tabs 2-16 are the member channels, Tab 1
the silent template, channel 1 the master channel).

## Why ReaNvelopes does not cover it

ReaNvelopes writes bend, channel pressure and CC74, but one curve stretched over the whole time range, on one channel.
MPE needs the opposite: many short gestures, each tied to one note and its channel. Six things are missing there.

1. Per-note targeting. A message must go on the channel of its note and only within that note's length.
2. Channel allocation. Overlapping notes need different channels from 2 to 16 (channel 1 is the master channel). The
   Channel note value in ReaNvelopes maps a curve to a channel range, so two overlapping notes can share a channel.
3. A gesture per note. Each note needs its own curve over its own length, optionally varied per note by a seed
   (his composing habit: deterministic variation, not live randomness).
4. Bend depth in semitones. The bend is written over its full 14-bit range, so the depth depends on the instrument's
   setting (plus or minus 48 for MPE member channels, plus or minus 2 for an ordinary channel). Needs a "plus or minus N
   semitones" control that assumes the instrument's range.
5. Reset between notes. The bend returns to centre and the pressure to zero when a note ends, before another note
   reuses the channel; also a bend set before the note-on, so a note never starts with the previous note's bend.
6. Setup messages. The MPE configuration message and the pitch-bend range per channel (RPN 0) are not sent.
   An instrument already in MPE mode (ReaSamp) does not need them; others do.

## Design sketch

Input: the selected MIDI item (its notes) and a gesture source. The gesture source is a curve, drawn or generated, that
is stretched over each note's length, with a per-note variation (seed, depth, optional offset in time or value). The
curves come from ReaNvelopes (DRAW, OSC, RAND) or ReaAC's modules, through the shared curve format
(`{lengthBeats, hold, points{t,v}}`), so MPEcreator needs no generators of its own.
Steps: allocate channels round-robin for overlapping notes (reuse a channel only after its note has ended plus a
release gap), write the reset events, write the three gestures per note, optionally write the setup messages at the
start of the item. Pitch bend, pressure and slide can each be switched on or off.
The channel allocation and the note plus bend logic exist already in ReaComposeM's `lib/MicroTuning.lua` (also embedded
in ReaMaskM); that is the place to start from, as with the Scala scale support. It also fits microtonal work: a note
plus a bend on its own channel is how a microtone is played in MPE.

## Where it fits in the family

Perspective 2, creating data (see `CLAUDE.md`, "How the family fits together"): ReaAC makes notes, ReaNvelopes makes
curves, MPEcreator puts the two together into an MPE performance, ReaShapeM finetunes a selection afterwards, ReaSnap
recalls the state. The gesture gap is the one the family section already names for MPE.

## Open questions for him

Same gesture for every note or a seeded variation. Which of bend, pressure and slide are written together. Whether the
channel allocation should follow pitch order, note order or a random seed. Whether it is a tool of its own (script in the
ReaSnap launcher, own ReaImGui window) or a Target inside ReaNvelopes after all. Whether notes already on channels 2-16
are kept or re-allocated.

## What MPElodyne teaches (21 October 2026)

tjingboem found MPElodyne, an MPE note-expression editor for REAPER made by someone else (`MPElodyne.lua`, `mpelodyne_core.lua`,
`test_mpelodyne_core.lua`; ReaImGui; no licence text in the files we saw). It is an editor: the user draws bend, pressure and
timbre by hand on a piano roll. MPEcreator is a generator. So the interface is not a model, but the way it handles the MIDI is.
Take the ideas, write our own code, and ask the author before copying anything (the licence is unknown).

**The five MPE dimensions.** MPE has five: strike (note-on velocity), glide (pitch bend), slide (CC74), press (channel pressure)
and lift (note-off velocity). MPElodyne does glide, slide and press only; its note-offs are written with velocity 0, so lift is
never touched. MPEcreator can be the whole set: ReaNvelopes already writes strike and lift (`commitNoteValue`), and ReaSamp's
Tilt reads lift. That is the reason to build it as its own tool.

**Read and write the raw event stream.** It uses `MIDI_GetAllEvts` and `MIDI_SetAllEvts` (the packed `i4 B s4` records), not
`MIDI_InsertCC` per event. One atomic write per edit, REAPER sorts once, the events can carry a CC shape in the flags byte
(`(flags & 0x0F) | 0x10` = linear), and the old stream is kept so the write can be undone or restored if REAPER refuses it.
ReaNvelopes' `commitCC` inserts one event at a time and then searches them again to set the shape; the raw route is simpler and
faster for thousands of events.

**Ownership by channel and time.** An expression event belongs to the note that sounds on its channel at that moment. This only
works when a channel never carries two notes at once, so the tool diagnoses same-channel overlaps and refuses to write until they
are fixed. MPEcreator writes this ownership itself, so it can guarantee it, and it should still re-read its own output, rebuild
the model and abort if the result is ambiguous (MPElodyne does exactly that after every move, resize and add).

**Channel allocation is a conflict graph.** Two notes conflict when they overlap or merely touch (start <= other end), because a
release tail or a leftover bend on the channel would bleed into the next note. Each note gets a random free channel from 1 to 15
(seeded linear congruential generator, so the same seed gives the same result); notes that are not being changed keep their
channel and count as conflicts; if more than 15 are needed it stops with a message. This replaces my round-robin sketch. New
single notes take the lowest channel that avoids overlaps and the neighbours before and after in time.

**Order at the same tick matters.** It packs events with priorities: the last value of a note's curve (20), note-off (30),
leftover resets (40), setup events before a note-on (40), note-on (50), everything inside the note (60). REAPER sorts an expression
event at a note's exact start before the note-on, so the reader claims such events for the upcoming note unless an outgoing note on
the same channel still owns that tick. Pitch bend, pressure and slide are written before the note-on (so the note starts at the
right value), and pressure and slide are written again right after the note-on (instruments that reset per note at note-on).
This replaces my "reset between notes" point: it does not reset after a note; it holds the final value through the note-off and
makes every new note set its own start values before it begins. That is cheaper and avoids stray events in the next note.

**Defaults for a new note.** Pitch bend centred (8192), pressure 100, slide 100, written as lines over the whole note. Pitch is
stored as semitones relative to the note, scaled by the bend range (default plus or minus 48); the conversion is asymmetric
(8191 up, 8192 down) so that centre is exactly 8192.

**Pressure format.** It detects whether the item uses channel pressure (D0) or polyphonic pressure (A0) and writes in the same
format. MPEcreator should offer both, channel pressure the default.

**Microtuning as a base offset.** Tunings are twelve cents offsets per pitch class (Pythagorean, 5-limit just, quarter-comma
meantone, two maqam approximations) plus a key. A tuned note is the MIDI pitch plus a starting bend equal to its offset, and every
gesture is added on top of that. This is the same idea as ReaComposeM's MicroTuning (note plus bend on an MPE channel) and should
be one shared piece of code. It also warns the user when a note sounds off 12-TET.

**Modulation stack, the part to copy as a concept.** Layers of sine, square, saw and triangle add together; Curve Up and Curve
Down layers shape either the amplitude or the rate of what came before. Rate is integrated (the phase is the integral of the
rate curve), so a rate curve gives a continuous speed-up or slow-down without clicks (progressive vibrato is a preset). The depth
and the rate are measured in cycles across the note, not in Hz, and the first and last 3.5 percent taper to the baseline so the
modulation starts and ends at the note's own value. The number of samples follows the highest rate (rate times 32 plus 1, between
65 and 1025). For MPEcreator this is the gesture engine: vibrato, swell, scoop, flutter, per note, with a seed for variation.
ReaNvelopes' OSC and RAND already produce such curves; the new part is stretching one gesture over each note's length with
a per-note variation, and the integrated rate.

**Smaller points worth keeping.** Generated events must not inherit the note's selection flag (bit 0). Loop-aware items: a
looped source repeats its notes and a take offset moves the visible window, so positions must be mapped through the item
(`B_LOOPSRC`, `BR_GetMidiSourceLenPPQ`). Audition: it sends bend, pressure and slide first and then the note-on with
`StuffMIDIMessage` on a spare channel, so a gesture can be heard before it is written. The core is a separate pure-Lua file
tested with plain `lua5.4` (about 250 assertions), the same split we use for the curve engine; MPEcreator should be built the same way.

**What MPEcreator does differently.** It does not edit by hand. It takes a source for each dimension (a curve from DRAW, OSC
or RAND, a ReaAC module, a preset gesture), a depth in semitones for glide, and a seed for the variation per note. It would write
glide, slide, press and lift for every note of the item, and strike as well if wanted, with the channel allocation above. Lift
needs one more thing MPElodyne does not have: the note-off velocity edit (`string.pack("<i4Bs4")` on the raw stream, as in
ReaNvelopes' `commitLift`).

Revised build order if he says go: (1) pure core: parse and write the raw stream, ownership, conflict-graph channel allocation,
tests; (2) the five dimensions with flat defaults; (3) the gesture engine (vibrato and swell first); (4) seeded variation per
note; (5) microtuning offsets; (6) audition. The licence question about MPElodyne stays open until he has asked the author.

## The rules model (his idea, 21 October 2026)

Purpose, in his words: a Lua script that highly automates adding MPE to notes. As with the red diamonds in ReaShapeM, the tool detects
the moment of each note-on and reads the channel, pitch and duration. The user does not draw; he sets rules, and the tool applies
them to every note. Each MPE dimension (glide, slide, press, lift, and strike if wanted) has its own rule set.

His first rules, for pitch bend:
1. Where in the note: a range between note-on and note-off (for example 20 to 80 percent of the note).
2. How many bends per note: a range, 1 to 5.
3. Which direction: a mask (up, down, or both; possibly a pattern for several bends).

His decision that settles the release: at note-off the bend stays as it is at that moment. Nothing is reset at the end of a
note, and the release of the sound simply rings out at that pitch.

Consequence to keep in mind (from MPElodyne's handling): the bend stays on the channel, so the next note on that channel must set
its own start value before its note-on, or the channel allocation must not reuse a channel while its bend is off centre. The
simplest rule is that every note writes its own start values just before its note-on. The same holds for press and slide.

Rules Claude would add to the list (to be accepted or dropped by him):
- Amount: a range in semitones (for example 0.5 to 3), so a bend stays inside the instrument's bend range.
- Shape of each move: linear, ease in or out, S-curve, or a step (a jump).
- Duration of each move as a fraction of the note, and whether moves overlap or follow each other.
- Several bends: do they accumulate from where the last one stopped, or does each one aim at its own target.
- Start value: centre, or the microtuning offset of the note.
- Snap: free, semitone, or scale degree (or the microtuning scale).
- Probability: the share of notes that get a bend at all, and a minimum note length below which nothing is added.
- Filters: apply only to notes in a pitch range, a channel, a velocity range, or a length range.
- Seed: the same seed gives the same result, so a variation can be recalled (ReaSnap).
- Control by a curve: any rule value can follow a ReaNvelopes curve over the item (read at each note's start), the same way the
  Note value target works. That keeps his principle: external control over internal randomness.
- Glide to the next note: a bend that ends on the pitch of the following note when it lies inside the bend range (a legato slide).
- Lift is one value at the note-off, so its rules reduce to a range and a direction relative to the note-on velocity.

Step zero for non-MPE material: bends are per channel, so notes on one channel must first be spread over channels 2 to 16
(the conflict-graph allocation above) before any rule is applied.

Open questions for him: whether the red diamond note reader of ReaShapeM can be shared as one library for both tools; which bend
range ReaSamp's MPE Tabs use (the tool must assume the same one); whether the direction mask is three states or a pattern per bend.

### His answers to the open questions (21 October 2026)

1. The note reader of ReaShapeM (the red diamonds: note-on moment, channel, pitch, duration) becomes one shared library used by
   ReaShapeM and MPEcreator. Not yet looked at: how ReaShapeM implements it today, and whether it can be lifted out without
   changing ReaShapeM's behaviour.
2. The bend range is at most 96 semitones. Consequence: 14 bits over plus or minus 96 semitones is 1.17 cents per step (at 48 it
   is 0.59 cents, at 2 it is 0.024 cents), so the amount limits are set in semitones and converted with the range, and very small
   microtonal moves need a smaller range.
3. Direction is not a mask but two probability envelopes running in parallel over the whole item: one gives the chance that a
   note gets a bend up, the other the chance that it gets a bend down, both as a percentage over time. At 1 every note has that
   curve. So the direction of the whole piece is shaped in time by two curves, not set once.

Open inside answer 3: how the two probabilities combine. Options: (a) one roll per note: up if the roll falls under P(up), down if
it falls in the next P(down) band, nothing otherwise, and when the sum is above 1 the two are scaled to fit; (b) two independent
rolls, so a note can get both an up and a down (with both at 1, every note gets both); (c) one roll per bend when a note has
several bends. Claude's recommendation: (c) with (a) as the rule for each bend, since it works with the 1 to 5 bends rule and is
repeatable from the seed. Where the two curves come from: drawn inside MPEcreator with the DRAW code of ReaNvelopes (a shared library),
read at each note's start (or at each bend's moment), like the Note value target.

### Rules are masks (his statement, 21 October 2026)

Still the thinking phase; the combining rule for the up and down probabilities is not decided. One principle is: the rules must
work like the masks of ReaMaskM, where a mask changes its range over time. A rule is therefore never one fixed number or one
fixed range; it is a pair of envelopes over the item, a minimum and a maximum, and the range between them can open, close and
move along the piece.

How ReaMaskM does it (read from its `CLAUDE.md`, 21 October): every parameter is a draggable envelope, in four families
(Pitch, Time, Dynamics, Channels). A range is two envelopes, named Min and Max with "(mask)" in the name (Min Channel (mask),
Max Channel (mask), Minimum Pitch (mask), Maximum Pitch (mask)). The envelopes are sampled across the item. When the minimum
ends up above the maximum it collapses down to the maximum (it is not swapped), and this holds in the maths whichever envelope is
dragged higher on the canvas. A value outside a hard bound is skipped, not clamped. A live number appears in a tooltip while
dragging (ReaMaskM has it only for Channel, and the lack of numbers elsewhere was noted as a gap). Whole settings are saved as
`.mask` presets, and AUDITION previews the result by running the real writer.

For MPEcreator that gives, per dimension (Glide, Slide, Press, Lift) a family of masks: where in the note the move starts (min and
max, in percent of the note), how many bends (min and max, 1 to 5), the amount (min and max, in semitones), the length of each
move, the chance of up and the chance of down (single envelopes, as he described), and the filters. At each note-on the tool reads
every envelope at that moment and draws the note's values from the range it finds there. Same collapse rule for every pair,
numbers shown while dragging from the start, presets in the style of `.mask`, and an audition that runs the real writer.
Possible reuse: the envelope canvas, the min and max collapse, the preset system and the audition of ReaMaskM, rather than writing
them again; and ReaMaskM's MicroTuning with its channel allocator (channel 1 reserved, 2 to 16 cycled) for step zero.

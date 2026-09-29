***Program by NaomiK + LucasISO , FluoroPolymers***
# MFX-lib

MidiMath v0.1.1

A small desktop tool that converts between two music formats:

MIDI (.mid, .midi): the standard note-data format.
MIDI File Express MFX (.mfx): a plain-text format that stores music as curves (pitch and velocity over time) instead of note events.

Depending on what you open, it does one of two things:

You open...	It does...
an .mfx file	Converts it straight to a MIDI file ("name"_reconstructed.mid)
a .mid file	Analyzes it, splits it into voices, and shows an interactive piano-roll plot with buttons to export as MIDI or MFX
you'll prob need :
- Python 3
- mido, numpy, matplotlib
- tkinter (bundled with most Python installs) , if not you're a bum
  ```pip install mido numpy matplotlib```

Run the script duh
A file picker opens. Choose a .mid or .mfx file.
If you chose an .mfx: a _reconstructed.mid file is saved next to it and a popup confirms.
If you chose a .mid: a plot window opens.
- The x-axis is time in seconds and the y-axis is MIDI pitch.
- Each note is a horizontal bar, colored by voice.
- Export Reconstructed MIDI saves <name>_reconstructed.mid
- Export .mfx saves "name".mfx
  
  # The MFX format (ie "*Für Elise*" by our goat Beethoven)
```
[HEADER]
title = "fur elise (original).mid"
bpm = "120"
time_signature = "4/4"
[/HEADER]

[TRACK]
name = "Voice_1"
channel = "0"
program = 0
[/TRACK]

{CURVE}
type = "PITCH"
interpolation = "STEP"
points = {
  (3.673,69,0.0,0.0),
  (4.286,69,0.0,0.0),
  ...
}
{/CURVE}

{CURVE}
type = "VELOCITY"
...
{/CURVE}
```
Header: title, tempo, and time signature.
Track: name, MIDI channel (0-based), and instrument program. The included Für Elise file has 4 tracks (Voice_1 to Voice_4).
Curves: these come after the [/TRACK] line, and each belongs to the most recent track. Each track has two:
PITCH: the note number (69 = A4, 72 = C5) at each time.
VELOCITY: how hard the note is played (0 to 127).
Points: each is (time_in_seconds, value, 0.0, 0.0). The last two numbers are unused in this file.
STEP interpolation: a value holds until the next point changes it, like a staircase.


# How it works , I have no fucking idea but I guess it does work

***MFX → MIDI***

parse_mfx reads the text file line by line, tracking whether it is inside a header, track, curve, or points block. It builds a dictionary of the header plus a list of tracks, each with its curves.
reconstruct_voices_from_mfx turns the curves back into notes, using the velocity curve as a gate:
Velocity goes from 0 to a positive value: a note starts.
Velocity goes back to 0: the note ends.
The note's pitch is the most recent pitch-curve value at or before the start time.
Velocity changes while a note is sounding do not create a new note.
A note still open at the end is closed at the last timestamp.
reconstruct_midi builds the MIDI file with mido: a tempo track plus one track per voice. Seconds are converted to ticks, and events are sorted so a note-off comes before a note-on at the same tick.


MIDI → plot → MFX
analyze_midi reads all note-on/note-off pairs and converts their times to seconds. It splits the notes into up to 4 voices (DEFAULT_NUMBER_OF_VOICES) with a greedy rule: each note goes into the first voice that is free (its previous note has ended). If every voice is busy, it goes into the voice that frees up soonest. It also builds a step curve of pitch over time for each voice.
create_interactive_plot draws the bars and step lines with Matplotlib and adds the two export buttons.
export_mfx writes each voice as a track with a PITCH curve and a VELOCITY curve. Velocity is written as (start, velocity) then (end, 0), exactly the on/off pattern the parser reads back.

Because the exporter writes the same format the parser reads, you can round-trip: MIDI → MFX → MIDI.

where it becomes shitty as hell :
Voice count is fixed at 4. Change DEFAULT_NUMBER_OF_VOICES at the top of the script. With more than 4 simultaneous notes, notes get squeezed into voices and can overlap.
Export .mfx always writes bpm = 120, even if the MIDI has a different tempo. Note times are saved in seconds, so timing is still correct, but the tempo label is not.
Tempo detection is simplistic. It uses a single tempo and ignores tempo changes. The value used may come from the last track that has a tempo event, not the first.
Instruments are lost. program is always written as 0 (piano), and program changes, pedal, and other controller data are not read.
Original tracks are flattened. The MIDI's own track structure is merged and re-split into voices.
Only PITCH and VELOCITY curves are used. Other curve types in an MFX file are ignored.
Opening an .mfx goes straight to conversion with no plot. The plot only appears for MIDI files.

What we'll do once exams are over ? 
- sleep (that'd be a great start)
- make it open a 2nd tkinter window that actually allows you to tweak a few things for the .mfx such has max voice count , standarized bpm and some other musical shit I guess , ask Naomi about that not me

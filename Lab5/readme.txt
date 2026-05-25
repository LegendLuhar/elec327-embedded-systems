Lab 5 Write-Up - Rahul Kishore

State Machine and Code Architecture

The program uses a decoupled state machine design with two high-level modes: SONG mode and BUTTON_TONE mode. 
On startup, the system enters SONG mode and continuously plays “Mary Had a Little Lamb.” 
The melody uses the correct three notes (C4, D4, E4) in the proper pattern. 
When any debounced button press is detected, the system immediately exits SONG mode and transitions to BUTTON_TONE mode. 
There is no return to SONG mode unless the board is reset.

Song playback is controlled using a note index, a tick counter, and a gap flag. 
Each note is followed by an 80 ms silent pause so notes are clearly distinct. The note durations include this pause. 
A quarter note is 700 ms total, a half note is 1400 ms total, and a whole note is 2800 ms total, 
ensuring that each duration is exactly twice the previous one (including the gap), satisfying the required timing relationships.
The melody is stored in a simple array of {load_value, duration_ticks} structures. 
Changing the tune only requires editing this array, so the music can be easily modified. 
A load value of 0 represents a rest.

Each of the four buttons has its own debounce state (30 ms debounce at 10 ms sampling, active-low inputs). 
In BUTTON_TONE mode, each button produces a distinct frequency (G6, E6, C6, G5). Pressing any button stops the song immediately. 
A tone plays while a button is held, and PWM output is disabled when all buttons are released, ensuring tones stop correctly.

Overall, the architecture cleanly separates mode control, debounce logic, song playback state, and PWM hardware control, 
resulting in modular and predictable behavior that satisfies all rubric requirements.
ELEC 327 — Lab 6 Writeup - Rahul Kishore
SPI LED Control + State Machine Architecture
1. State Machine Architecture
The firmware is organized as a two-mode finite state machine, implemented in state_machine.c/.h. The machine has one top-level mode variable (top) that selects between TOP_MUSIC and TOP_BUTTONS. All state is captured in a state_t struct that is passed by value into TickStateMachine() each timer tick, ensuring there are no hidden globals that could corrupt state between ticks.

1.1  Modes
TOP_MUSIC is the entry mode. On every TIMG0 tick the helper AdvanceMusicTick() is called. It maintains two sub-state variables:
•	tick — counts TIMG0 ticks elapsed within the current note's total time slot.
•	in_gap — a flag that is 0 while the note's tone is sounding and 1 during the short inter-note silence appended to every note.
When tick reaches the sound duration for the current note's length, the buzzer is silenced and all LEDs are turned off (entering the inter-note gap). When tick reaches the total slot duration, the note index advances (wrapping around to replay the song), tick resets to zero, in_gap clears, and the next note's buzzer period and LED packet are loaded immediately so the new note starts on the very next tick.

TOP_BUTTONS is entered the moment UpdateDebounce() reports that any button is confirmed pressed while the machine is in TOP_MUSIC. The transition is one-way — the board stays in button-feedback mode for the remainder of the power cycle. In this mode, SetButtonOutput() inspects the four pressed[] flags (checked in priority order SW1–SW4) and activates the matching LED packet and buzzer tone. When all buttons are released, the buzzer is disabled and all LEDs are turned off.

1.2  Debouncing
Button state is read from GPIOA->DIN31_0 on every timer tick. Each button has its own 8-bit saturation counter (db[]) that increments when the GPIO reads pressed (active-low, so inverted in software) and decrements otherwise. A button is considered confirmed pressed only when its counter reaches DEBOUNCE_THRESH (10 ticks, approximately 6.4 ms). This eliminates contact bounce without requiring any additional timers or interrupts.

1.3  Timing
TIMG0 is clocked from LFCLK (32,768 Hz) with a LOAD value of 20, producing an interrupt period of approximately 0.641 ms. One quarter-note spans QUARTER_TOTAL = 800 ticks (~513 ms). Within each slot, the note sounds for QUARTER_SOUND = 750 ticks and the inter-note silence fills the remaining 50 ticks. Half-notes and whole notes scale linearly by multiplying QUARTER_TOTAL.

2. LED Color Design
The APA102C LEDs are driven over SPI0 using the standard start-frame / per-LED-frame / end-frame protocol. Each 16-bit LED frame is split across two SPI words: word1 carries the brightness field and the blue channel, word2 carries green (high byte) and red (low byte). Twelve 16-bit words are sent per update: two start words (0x0000), eight LED words (two per LED × four LEDs), and two end words (0xFFFF).

2.1  Note Colors (TOP_MUSIC)
Three pure RGB primary colors were assigned to the three notes in the melody. Using primaries means the colors are maximally distinct at any brightness level — there is no two-channel mix that could be mistaken for another note.

Note	Color	LED Word1 (blue)	LED Word2 (green, red)
E6 (~1319 Hz)	Red	0xE800	0x00FF (red=255)
D6 (~1175 Hz)	Blue	0xE8FF (blue=255)	0x0000
C6 (~1047 Hz)	Green	0xE800	0xFF00 (green=255)
Rest / Gap	Off (all dark)	0xE000	0x0000

The mapping follows an intuitive pitch-to-warmth gradient: the highest note (E6) is red (warm / high energy), the middle note (D6) is blue (cool / neutral), and the lowest note (C6) is green (calm / low). All four LEDs display the same color simultaneously so the entire board changes as one unit, reinforcing the sense that the color represents the current musical pitch. During every inter-note gap all LEDs are extinguished, giving a clear visual beat between notes.

2.2  Button Colors (TOP_BUTTONS)
In button-feedback mode each LED is individually mapped to one button, following the classic Simon color scheme. Only the LED geometrically nearest the pressed button lights up; the other three remain dark.

Button	LED Index	Color	Simon Tone
SW1 (top-left)	LED 0	Green	G6 (~1568 Hz)
SW2 (top-right)	LED 1	Red	E6 (~1319 Hz)
SW3 (bot-left)	LED 2	Yellow	C6 (~1047 Hz)
SW4 (bot-right)	LED 3	Blue	G5 (~784 Hz)

Yellow is achieved by mixing green=200 and red=255 (blue=0), which produces a visually saturated warm yellow at the brightness level used (0xEC = brightness 12). This color is unambiguous against the solid green (SW1), solid red (SW2), and solid blue (SW4) of the other buttons.

3. Main Loop and ISR Interaction
Two ISRs are active simultaneously: TIMG0_IRQHandler() sets timer_wakeup = true; SPI0_IRQHandler() clocks out successive words of the current LED message and sets spi_wakeup = true when each word is consumed by the TX FIFO. The main loop distinguishes the two wakeup sources using the public flag variables exposed by timing.h and leds.h:
•	If timer_wakeup is set: the state machine is ticked, current_led_packet is updated, and SendSPIMessage() is called. A spin-wait ensures any in-flight SPI transmission completes first (in practice this is rare since the SPI transfer takes ~96 µs and the timer period is ~641 µs).
•	If only spi_wakeup is set (SPI mid-transfer): the flag is cleared and the loop returns to WFI immediately without ticking the state machine.

This separation ensures that the state machine advances at a deterministic rate driven solely by the hardware timer, while the SPI peripheral is free to run at its own pace in the background via interrupts.

4. Module Breakdown
The firmware is split into six focused modules:
•	buttons.c/.h — GPIO IOMUX init, pin definitions for SW1–SW4 (PA23–PA26).
•	buzzer.c/.h — TIMA1 PWM init, SetBuzzerPeriod(), EnableBuzzer(), DisableBuzzer(). Note period constants for all tones used.
•	timing.c/.h — TIMG0 LFCLK timer init, SetTimerG0Delay(), EnableTimerG0(), timer_wakeup flag.
•	leds.c/.h — SPI0 init, interrupt-driven SendSPIMessage(), spi_wakeup flag.
•	state_machine.c/.h — All song/button logic, LED packet definitions, current_led_packet pointer.
•	lab6.c — main(): initializes all modules, runs the WFI loop.

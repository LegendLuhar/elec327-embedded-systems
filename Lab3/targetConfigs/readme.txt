Rice ELEC 327
Lab 3 – Processing GPIO Input and Setting the Clock

Overview
In this lab we extended the Lab 2 clock implementation by adding user interaction using a pushbutton connected to PB8. The clock is now controlled by a hierarchical finite state machine whose next state depends on both the current state and the button input.

What We Implemented

Multiple Operating Modes
• Normal Clock Mode
• Hour-Set Mode
• Minute-Set Mode
• Brightness-Set Mode (extra credit)

Button Handling
• Long press defined as holding ≥ 1 second
• Short press defined as ≥ 5 ms and < 1 second
• Debouncing handled inside the state machine using tick counters
• Short presses trigger actions on release
• Long press fires once per hold

Mode Transitions
Normal → Hour-Set → Minute-Set → Brightness-Set → Normal

Visual Feedback
• Hour flashes in Hour-Set mode
• Minute flashes in Minute-Set mode
• Both flash in Brightness-Set mode
• Flashing implemented using a 1 Hz software counter
• No blocking delays used

Time Adjustment
• Hour increments with wrap-around
• Minute increments in 5-minute steps with wrap-around
• Brightness cycles through 15 PWM levels

Power Efficiency
• TimerG0 runs from LFCLK
• CPU sleeps in standby between interrupts
• PWM implemented using software counter at 128 Hz

State Machine Design

The state struct stores:
• Current time
• Mode
• Button history
• Flash counters
• Clock counters
• PWM state

GetNextState computes the next state using current state and input.
GetStateOutput computes LED outputs purely from the state.

No global variables are used inside the state machine logic.

Assumptions

• Valid button presses are ≥ 5 ms
• No glitching longer than debounce threshold
• Minimum brightness level is visible in low light

All functional checklist items were verified.
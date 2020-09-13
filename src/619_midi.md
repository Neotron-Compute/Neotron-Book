# MIDI

MIDI is basically an opto-isolated UART based protocol running at a fixed 31,250 bps. The MIDI In port is connected to UART RX via an opto-isolator, and the MIDI Out port is driven from UART TX via a 5V level shifter (two back-to-back inverters from a hex-inverter).

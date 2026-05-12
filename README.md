# Group13
Brain Healer — FPGA Sequence Memory Game
Overview
Brain Healer is a hardware-implemented sequence memory game built on the Alchitry Au FPGA (Xilinx Artix-7, 100MHz) using Lucid HDL. Players watch a sequence of directional arrows displayed on an 8×8 LED matrix and must reproduce the sequence using physical buttons. Each level adds one more arrow, and a reverse mode challenges players to input the sequence backwards.

Hardware Setup
ComponentConnectionAlchitry Au (100MHz)Main FPGA boardIO ShieldButton/LED interfaceMAX7219 8×8 LED matrixio_segment[4] = data, [3] = load, [1] = clock2× 7-segment displayio_select, io_led[1][6:7], io_led[2][0:4]Breadboard buttonsUP=io_dip[1][2], DOWN=io_dip[1][0], LEFT=io_dip[1][3], RIGHT=io_dip[1][4]StartLEFT + RIGHT simultaneouslyResetAll 4 buttons simultaneously

Gameplay

Game displays a sequence of arrows (↑ ↓ ← →) on the LED matrix
Player must repeat the sequence using the buttons within a 10-second timer
Correct sequence → next level (sequence length +1, score +3 or +5)
Wrong input or timeout → Game Over (X displayed on matrix)
Reverse mode: if rev_flag = 1, player must input the sequence in reverse order (+5 points vs +3)


Architecture
Module Structure
alchitry_top          ← top-level hardware wiring
├── game_data         ← datapath (regfile + ALU + RNG)
│   ├── game_regfile  ← 8×32-bit register file
│   ├── game_cu       ← FSM control unit
│   ├── alu           ← arithmetic logic unit
│   └── random_number_generator
├── matrix_controller ← MAX7219 driver + bitmap rendering
├── display_control   ← combinational display logic
└── seg_mux           ← 7-segment multiplexer
Register File
AddressRegisterInitialR0score0R1timer10R2seq_len2R3display_idx0R4input_idx0R5latched_arrow0R6temp0R7rev_flag0
ALU Operations
alufnOperation6h00ADD6h01SUB6h1APASS A6h33CMPEQ6h3CMOD
asel / bsel Mux
asel[3]:
000=score, 001=timer, 010=seq_len, 011=const 10, 100=display_idx, 101=input_idx, 110=latched_arrow
bsel[3]:
000=rd2, 001=1, 010=2, 011=expected_arrow, 100=3, 101=5

FSM Design
States (12 total)
IDLE → INIT_SCORE → INIT_LEVEL → INIT_INPUT_IDX → INIT_TIMER
     → SHOW_SEQUENCE ⟲
     → WAIT_INPUT
     → CHECK_MATCH → NEXT_ARROW → WAIT_INPUT
                  → LEVEL_CLEAR_SCORE → LEVEL_CLEAR_SEQLEN → INIT_LEVEL
                  → GAME_OVER → IDLE
FSM Minimization
Original design had 30+ states. Reduced to 12 through:
OriginalOptimized10 states for sequence display (shift reg per arrow)1 SHOW_SEQUENCE state with internal show_timer[27]4 separate Check states per button1 CHECK_MATCH with input_valid signal3 states for timer (check/branch/decrement)Inline condition in WAIT_INPUTDedicated reverse statesCombinational exp_shift with rev_flagALU operations causing combinational loopsDirect Lucid operators for comparisons, ALU for arithmetic only

Key Design Decisions
Why >> instead of ALU shift?
Arrow extraction uses direct bit shifting:
lucidcurrent_show_arrow = (seq.q >> show_shift) & 2b11
expected_arrow     = (seq.q >> exp_shift)  & 2b11
The ALU is occupied with arithmetic operations (ADD/SUB/PASS) on a per-state basis. Arrow extraction must be continuously valid across multiple states, so it is implemented as combinational logic outside the ALU.
Avoiding combinational loops
CHECK_MATCH uses direct Lucid comparison instead of ALU CMPEQ:
lucidif (rd_latched[1:0] == expected_arrow) { ... }
Routing this through the ALU would create a feedback loop: CU reads ALU output → ALU depends on CU control signals → loop.
Random sequence generation
LFSR-based RNG (random_number_generator, SIZE=30) clocked at ~95Hz via slow_clk_ctr(#DIV(20)). New sequence loaded on each seq_load pulse. rev_flag = rng.out[0] determines reverse mode per level.

Version History
VersionDescriptionv1Basic FSM, hardcoded sequence, no regfilev2game_regfile structure, button_conditioner + edge_detectorv3show_timer internalized, RNG integrated, rev_flag addedv4ALU fully integrated, asel[3]/bsel[3] mux expanded, score +3/+5

Build
Tested on Alchitry Labs 2. Timing constraint: WNS = -3.577ns (functional at 100MHz).
Tools: Alchitry Labs 2, Vivado (Xilinx Artix-7)
Language: Lucid HDL
Board: Alchitry Au + IO Shield

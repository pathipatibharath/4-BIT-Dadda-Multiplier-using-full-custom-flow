4-Bit Dadda Tree Multiplier (Full Custom Flow)
A 4-bit × 4-bit unsigned multiplier built with a Dadda tree reduction scheme, designed end to end in a full-custom flow using Cadence Virtuoso and the gpdk090 (90 nm) PDK. Every logic gate is drawn at transistor level, the adders are built hierarchically from those gates, and the final design is verified with Spectre transient simulation and laid out in Virtuoso Layout Suite XL.
---
Design Summary
Item	Value
Function	P[7:0] = A[3:0] × B[3:0] (unsigned)
Architecture	Dadda tree (AND-array + HA/FA reduction)
Technology	gpdk090, 90 nm CMOS
Supply voltage	1 V
Transistor size	W = 120 nm, L = 100 nm (both NMOS and PMOS)
Partial-product gates	16 AND gates
Adders	4 half adders + 8 full adders
Total instances in `dadda_algorithm`	28 (16 AND + 4 HA + 8 FA)
Pins / Nets	16 pins / 48 nets
Tools	Virtuoso Schematic Editor, ADE L, Spectre, Virtuoso Visualization & Analysis XL, Layout Suite XL
Design Hierarchy
```
inverter
   ├── and_gate
   ├── or_gate
   └── xor_gate
          └── half_adder (XOR + AND)
                 └── full_adder (2 × HA + OR)
                        └── dadda_algorithm (16 AND + 4 HA + 8 FA)
                               └── dadda_algorithm_tb (testbench)
```
Pin Description
Pin	Direction	Description
`A0` – `A3`	Input	Multiplicand bits (`A0` is the LSB)
`B0` – `B3`	Input	Multiplier bits (`B0` is the LSB)
`p0` – `p7`	Output	Product bits (`p7` is the MSB)
---
1. Theory: Dadda Tree Multiplier
1.1 Steps of Binary Multiplication
A binary multiplier has three steps:
Partial product generation: every bit of A is ANDed with every bit of B. For 4 bits this gives 16 partial products.
Partial product reduction: the partial products are compressed column by column until only two rows remain.
Final addition: the last two rows are added to produce the product.
1.2 Array, Wallace and Dadda Multipliers
In an array multiplier the reduction is done row by row, so the delay grows linearly with the number of bits. Tree multipliers reduce all columns in parallel using half adders (HA) and full adders (FA) as counters. A full adder takes 3 bits of the same weight and produces 1 sum bit (same weight) and 1 carry bit (next weight).
Feature	Wallace Tree	Dadda Tree
Reduction strategy	Reduces as much as possible every layer	Reduces only as much as needed for next height
Adders in reduction	More	Fewer
Final adder	Narrower	Wider
1.3 Dadda Height Sequence
The target column heights follow the sequence:
```
d(1) = 2,   d(j+1) = floor(1.5 × d(j))   →   2, 3, 4, 6, 9, ...
```
For a 4 × 4 multiplier the tallest column has 4 bits, so the reduction runs 4 → 3 → 2 (two layers), followed by the final addition.
1.4 Partial Product Naming Used in This Project
Each partial product `aᵢbⱼ` is given an index `m` as follows:
	a3	a2	a1	a0
b0	m3	m2	m1	m0
b1	m7	m6	m5	m4
b2	m11	m10	m9	m8
b3	m15	m14	m13	m12
where `m0 = a0b0`, `m1 = a1b0`, … , `m15 = a3b3`.
1.5 Column Heights
Weight	2⁰	2¹	2²	2³	2⁴	2⁵	2⁶
Number of bits	1	2	3	4	3	2	1
---
2. Algorithm (Hand Derivation)
2.1 Algorithm Sheet
![Algorithm sheet](algorithm/algorithm.jpeg)
The sheet above shows the partial product matrix and how it is reduced layer by layer.
2.2 Theory
The top of the sheet lists the 16 partial products arranged by bit weight. The columns hold 1, 2, 3, 4, 3, 2 and 1 bits, so the tallest column has 4 bits.
Each reduction layer groups bits of the same weight into HA (2 inputs) or FA (3 inputs) boxes. The sum output stays in the same column and the carry output moves to the next column.
Only the columns that exceed the next target height (3, then 2) are reduced. This is what keeps the Dadda adder count low.
The reduction is repeated until each column has at most 2 bits, and the last layer produces the final 8-bit result.
The count of boxes on the sheet gives the adder budget: 4 half adders and 8 full adders in total.
2.3 Reduction Stages
Stage	Height change	Adders used
Stage 1	4 → 3	HA1, HA2
Stage 2	3 → 2	HA3, FA4, FA5, FA6
Final addition	2 → 1	HA7, FA8, FA9, FA10, FA11, FA12
2.4 Output Mapping
Product bit	Source
`P0`	`m0` (direct, no adder)
`P1`	`S7`
`P2`	`S8`
`P3`	`S9`
`P4`	`S10`
`P5`	`S11`
`P6`	`S12`
`P7`	`C12`
---
3. Manual Design (Wiring Plan)
3.1 Wiring Plan
![Manual design](algorithm/manual_design.jpeg)
This drawing converts the algorithm into a connection plan before drawing the schematic. The partial products `m0` to `m15` enter from the top. Each HA/FA block is labelled with its sum (`S`) and carry (`C`) outputs, numbered so that every signal can be traced into the next layer.
3.2 Theory
Top layer (stage 1, 4 → 3): HA1 and HA2 compress the tallest columns.
Middle layer (stage 2, 3 → 2): HA3 and FA4 to FA6 combine the sums and carries from the top layer with the remaining partial products.
Bottom layer (final adder): HA7 and FA8 to FA12 add the remaining two rows, with the carry passing from one weight to the next (`C7 → C8 → C9 → C10 → C11 → C12`) until the product bits `S7` to `S12` and `C12` are produced.
Carries always move one column to the left (to the next higher weight), which is why `C12` becomes `P7`.
3.3 Why This Step Matters
Writing this plan first is what made the large `dadda_algorithm` schematic manageable, since each net in the schematic matches a label here.
---
4. Basic Gates (Transistor Level)
All gates use W = 120 nm and L = 100 nm devices from `gpdk090_nmos1v` and `gpdk090_pmos1v`.
4.1 Inverter
Schematic
![Inverter](schematics/inverter.png)
Theory
A CMOS inverter has one PMOS pull-up and one NMOS pull-down with gates tied together as the input. When the input is `0`, the PMOS conducts and the output is pulled to VDD. When the input is `1`, the NMOS conducts and the output is pulled to ground. Only one transistor conducts in steady state, so static power is almost zero. The inverter is also used inside the other gates to generate complemented signals.
Boolean Expression
```
a_bar = ¬a
```

4.2 AND Gate
Schematic
![AND gate](schematics/and_gate.png)
Theory
The AND gate is built from transmission gates (a parallel NMOS and PMOS pair) together with an inverter that generates the complement of the select signal. When the select input is `1`, the first transmission gate passes the other input to the output. When the select input is `0`, the second transmission gate connects the output to ground, so `out = 0`. Sixteen of these gates form the partial product array, and one more sits inside every half adder.
Boolean Expression
```
ab = a · b
```

4.3 OR Gate
Schematic
![OR gate](schematics/or_gate.png)
Theory
The OR gate uses the same transmission-gate idea. When the select input is `0`, the output follows the other input. When the select input is `1`, the output is connected to VDD (logic 1). An inverter generates the complement of the select signal for the transmission gates. The OR gate is used to combine the two carries in the full adder.
Boolean Expression
```
a+b = a + b
```

4.4 XOR Gate
Schematic
![XOR gate](schematics/xor_gate.png)
Theory
The XOR output is `a ⊕ b = a·b' + a'·b`. It is built with transmission gates and inverters: one signal is used to select whether the output follows the other input directly or its complement. XOR is the core of the adder, because the sum of two bits is exactly their XOR.
Boolean Expression
```
axorb = a ⊕ b = a·b' + a'·b
```

---
5. Adders
5.1 Half Adder
Schematic
![Half adder](schematics/half_adder.png)
Theory
A half adder adds two 1-bit numbers and has no carry input. It is built from one XOR gate and one AND gate. In the Dadda tree a half adder is used only where a column has to be reduced by exactly one bit.
Boolean Expressions
```
sum   = a ⊕ b
carry = a · b
```

5.2 Full Adder
Schematic
![Full adder](schematics/full_adder.png)
Theory
A full adder adds three 1-bit numbers (`a`, `b` and carry-in `c`). It is made from two half adders and one OR gate. The first HA adds `a` and `b`; the second HA adds that sum to `c`; the two HA carries are ORed to give the final carry.
Boolean Expressions
```
sum   = a ⊕ b ⊕ c
carry = a·b + c·(a ⊕ b)
```

---
6. Top-Level Multiplier
6.1 Schematic
![Multiplier schematic](schematics/dadda_mult.png)
Theory
This is the full 4-bit Dadda tree multiplier. The structure follows the manual design directly:
Top row: 16 AND gates generate the partial products `m0` to `m15` from inputs `A0–A3` and `B0–B3`.
Middle: half adders and full adders reduce the partial products layer by layer (4 → 3 → 2).
Bottom: the last row of adders produces the upper product bits.
Outputs: `P0` to `P7` form the 8-bit result.
The schematic has 28 instances (16 AND gates, 4 HAs and 8 FAs), 16 pins and 48 nets, matching the plan in the algorithm sheet.
6.2 Symbols
![Symbols](symbols/symbols.png)
Theory
Each cell has a custom symbol so it can be used hierarchically: inverter, AND, OR, XOR, HA, FA, and the 4-bit multiplier block with inputs `A0–A3`, `B0–B3` and outputs `p0–p7`. Each verified schematic is wrapped in a symbol and then reused as a black box in the next level up (bottom-up design).
---
7. Testbench and Simulation
7.1 Testbench
![Testbench](schematics/dadda_mult_tb.png)
Theory
The testbench `dadda_algorithm_tb` instantiates the multiplier symbol and drives each input (`A0–A3`, `B0–B3`) with a pulse voltage source. Each source has a different period (multiples of 10 ns), so many different operand combinations are applied in one run. Each output `p0–p7` is connected to an output pin for probing.
Testbench Parameters
Parameter	Value
Source type	Pulse voltage source
`v1` (low)	0 V
`v2` (high)	1 V
Rise time (`tr`)	20 ps
Analysis	Transient, 0 – 160 ns
Simulator	Spectre (via ADE L)
7.2 Input Waveforms
![Input waveforms](outputs/transient_inputs.png)
Theory
The eight inputs switch between 0 V and 1 V (logic 0 and logic 1). Because every input has a different period, the operands `A = A3A2A1A0` and `B = B3B2B1B0` take many different values over the 160 ns window. At any instant the expected product is `A × B`.
7.3 Output Waveforms
![Output waveforms](outputs/transient_outputs.png)
Theory
The outputs `p0` to `p7` form the 8-bit product and follow the inputs as expected for combinational logic.
The outputs swing between about 0 V and 1 V.
Short spikes appear at the switching instants. These are glitches caused by unequal path delays through the adder tree when many inputs change at once. They settle before the next transition.
Small deviations in the logic levels (for example on `p1`) come from the transmission-gate style gates.
`p0` has the shortest path, while `p7` has the longest delay, through the final adder chain.
Expected Values for Checking
To check a time instant, read `A` and `B` from the input plot and compare `p7…p0` with the expected product:
A	B	A × B	P7 P6 P5 P4 P3 P2 P1 P0
0	0	0	0  0  0  0  0  0  0  0
3	5	15	0  0  0  0  1  1  1  1
7	9	63	0  0  1  1  1  1  1  1
15	15	225	1  1  1  0  0  0  0  1
---
8. Layout
8.1 Layout View
![Layout](layout/dadda_mult_lay.png)
Theory
The layout was drawn in Virtuoso Layout Suite XL for the gpdk090 process. Standard CMOS layout layers are used: Nwell, Poly, Pimp/Nimp diffusion, Contacts, and Metal1 onward for routing, with vias between metal layers. Transistors for each gate are placed in rows, with PMOS in n-well regions and NMOS in the substrate, and the VDD rails run along the top of the design. Signals are routed on Metal1/Metal2 and above.
8.2 Verification Status
Check	Status
DRC (Design Rule Check)	☐
LVS (Layout vs. Schematic)	☐
PEX / post-layout simulation	☐
---
9. Repository Structure
```
.
├── README.md
├── LICENSE
├── algorithm/
│   ├── algorithm.jpeg
│   └── manual_design.jpeg
├── schematics/
│   ├── inverter.png
│   ├── and_gate.png
│   ├── or_gate.png
│   ├── xor_gate.png
│   ├── half_adder.png
│   ├── full_adder.png
│   ├── dadda_mult.png
│   └── dadda_mult_tb.png
├── symbols/
│   └── symbols.png
├── outputs/
│   ├── transient_inputs.png
│   └── transient_outputs.png
└── layout/
    └── dadda_mult_lay.png
```
---
10. How to Reproduce
Create a Virtuoso library attached to the gpdk090 technology.
Draw the `inverter`, `and_gate`, `or_gate` and `xor_gate` schematics (W = 120 nm, L = 100 nm).
Create symbols and build `half_adder`, then `full_adder`.
Build the `dadda_algorithm` schematic using the connection plan in `algorithm/manual_design.jpeg`.
Create `dadda_algorithm_tb` with pulse sources on A and B and run a 160 ns transient simulation in ADE L (Spectre).
Draw the layout of `dadda_algorithm` and run DRC and LVS.
---
11. Future Work
Test with more input combinations (for example, 7 × 9, 12 × 5, 15 × 1) and compare with expected products.
Measure propagation delay, power and area.
Complete DRC and LVS, and run post-layout (extracted) simulation.
Compare against a Wallace tree and an array multiplier.
---
License
Released under the MIT License. See LICENSE.
---
Author
P. Bharath Kumar <br>
Department of Electronics and Communication Engineering, IIITDM Kurnool
Under the guidance of: <br>
<Guide name> <br>
<Designation>, Department of Electronics and Communication Engineering, IIITDM Kurnool

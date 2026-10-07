# SECTION 2 — TUTORIAL SOLUTIONS (Tut-3 → Tut-7)

> **Scope:** every question in Tutorials 3–7, **including** sequential circuits (latches, flip-flops, counters, FSMs, state diagrams), and **excluding** register questions as you asked. Tut-7 Q4 (shift-register delay line) and Tut-7 Q5 (serial adder built on shift registers) are left out for that reason.
> **Checking:** every K-map answer below was cross-checked by an exhaustive Quine–McCluskey run, and every designed circuit was simulated over all input combinations.
> **Notation:** `'` = complement, `⊕` = XOR, `⊙` = XNOR, `Q+` = next state. In a state table, a cell like `01/0` means next state 01, output 0.

---

## QUICK REFERENCE — FLIP-FLOP TABLES (used in Tut-6 and Tut-7)

| Characteristic | D | T | JK | SR |
|---|---|---|---|---|
| Equation | `Q+ = D` | `Q+ = T ⊕ Q` | `Q+ = JQ' + K'Q` | `Q+ = S + R'Q` (SR = 11 not allowed) |

**Excitation tables** (what input causes a given transition):

| Q → Q+ | D | T | J | K | S | R |
|---|---|---|---|---|---|---|
| 0 → 0 | 0 | 0 | 0 | X | 0 | X |
| 0 → 1 | 1 | 1 | 1 | X | 1 | 0 |
| 1 → 0 | 0 | 1 | X | 1 | 0 | 1 |
| 1 → 1 | 1 | 0 | X | 0 | X | 0 |

- **D:** `D = Q+`, with no don't-cares. Easiest to fill in, often the most logic.
- **T:** T = 1 exactly when the bit **changes**. Ideal for counters.
- **JK:** every row has an X, which gives the most freedom and usually the least logic. Rule of thumb: when Q = 0 only J matters (J = Q+); when Q = 1 only K matters (K = (Q+)').

**Sequential design recipe** (Lec-20-21):
1. State diagram.
2. State table.
3. Assign binary codes.
4. Excitation table: for each flip-flop, fill in its inputs from the excitation table.
5. K-maps → flip-flop input equations and output equation.
6. Draw the circuit.
7. Check the unused states.

---

# TUTORIAL 3 — Subtractors, Adder/Subtractor, Scaling, Reading Verilog

## Tut-3 Q1 — Building a Faster Subtractor

### (a) Half-subtractor truth table and minimal SOP from first principles
- `X − Y` for 1-bit X, Y:
  - 0 − 0 = 0, no borrow.
  - 0 − 1: we must borrow 1 from the next column (worth 2), so the difference = 2 − 1 = 1, borrow = 1.
  - 1 − 0 = 1, no borrow.
  - 1 − 1 = 0, no borrow.

| X | Y | D | B |
|---|---|---|---|
| 0 | 0 | 0 | 0 |
| 0 | 1 | 1 | 1 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 0 |

- K-map for D:

| X \ Y | 0 | 1 |
|---|---|---|
| 0 | 0 | **1** |
| 1 | **1** | 0 |

  - Checkerboard: the two 1s are diagonal, not adjacent, so no grouping is possible. `D = X'Y + XY' = X ⊕ Y`.
- K-map for B:

| X \ Y | 0 | 1 |
|---|---|---|
| 0 | 0 | **1** |
| 1 | 0 | 0 |

  - A single isolated 1 → `B = X'Y`.

```
 X ──●───────────[XOR]──── D = X⊕Y
     │            │
 Y ──┼──●─────────┘
     │  │
     │  └────────┐
     └─|>o── X' ─[AND]──── B = X'Y
```

### (b) Algebraic proof: full subtractor = 2 half subtractors + OR
- HS1 on (X, Y): `D1 = X ⊕ Y`, `B1 = X'Y`.
- HS2 on (D1, Bin), computing D1 − Bin: `D2 = D1 ⊕ Bin`, `B2 = D1'·Bin`.
- **Difference:** `D2 = (X ⊕ Y) ⊕ Bin = X ⊕ Y ⊕ Bin` ✓ (the standalone FS difference).
- **Borrow:**
  - `Bout = B1 + B2 = X'Y + (X ⊕ Y)'·Bin`
  - `= X'Y + (XY + X'Y')·Bin` (XNOR expanded)
  - `= X'Y + XY·Bin + X'Y'·Bin`
  - `= X'(Y + Y'·Bin) + XY·Bin` (factor X' from the 1st and 3rd terms)
  - `= X'(Y + Bin) + XY·Bin` (redundant literal: `Y + Y'Bin = Y + Bin`)
  - `= X'Y + X'Bin + XY·Bin`
  - `= X'Y + Bin(X' + XY)`
  - `= X'Y + Bin(X' + Y)` (redundant literal: `X' + XY = X' + Y`)
  - `= X'Y + X'Bin + Y·Bin` ✓ (the standalone FS borrow).

### (c) Borrow-generate and borrow-propagate
- Rewrite: `B(i+1) = Xi'Yi + (Xi' + Yi)·Bi`, which has the form `Gi + Pi·Bi`.
- **`Gi = Xi'Yi`**: this bit position produces a borrow by itself (X = 0, Y = 1).
- **`Pi = Xi' + Yi`**: this bit passes an incoming borrow on (X = 0, or Y = 1).
- ⚠️ EXAM TRICK: `Pi = Xi ⊕ Yi` does **not** work here. At X = 1, Y = 1, Bin = 1: `1 − 1 − 1 = −1` needs a borrow, but `Xi ⊕ Yi = 0` would block it. The OR form gives P = 1 ✓.

### (d) Direct expression for B4
- Unroll `B(i+1) = Gi + Pi·Bi` four times:
  - `B1 = G0 + P0·Bin`
  - `B2 = G1 + P1G0 + P1P0·Bin`
  - `B3 = G2 + P2G1 + P2P1G0 + P2P1P0·Bin`
  - **`B4 = G3 + P3G2 + P3P2G1 + P3P2P1G0 + P3P2P1P0·Bin`**

**Key Logic & Takeaway**
- The half-subtractor borrow `X'Y` is asymmetric. That one fact drives Q1(b), Q1(c) and Q3(b).
- Borrow-lookahead has exactly the same structure as carry-lookahead; only G and P change (`X'Y`, `X' + Y`).

---

## Tut-3 Q2 — A Life-Counter Bug (overflow detective)

### (a) Bit-width and the failure condition
- **Try widths:**
  - 2-bit signed (−2…+1): the correct row "1 + 1 = 2" could not be shown → rejected.
  - **3-bit signed (−4…+3):**
    - 2 + 2 = 4 → wraps to 4 − 8 = **−4** ✓ (matches the log)
    - 3 + 1 = 4 → **−4** ✓
    - −3 − 3 = −6 → −6 + 8 = **2** ✓
    - −4 − 3 = −7 → −7 + 8 = **1** ✓
    - 1 + 1 = 2 ✓ and 2 − 1 = 1 ✓
  - 4-bit signed (−8…+7): 2 + 2 would show 4 (correct), but the log shows −4 → rejected.
- **The circuit is a 3-bit 2's-complement adder/subtractor.**
- **Failure condition:** overflow happens exactly when the two operands the adder actually adds (A and B for S = 0; A and B'+1 for S = 1) have the **same sign** and the result has the **opposite sign**. Mixed-sign operands can never overflow.

### (b) Overflow flag from existing signals
- **`V = C2 ⊕ C3`** = (carry into the sign bit) XOR (carry out of the sign bit). Generally `V = C(n−1) ⊕ Cn`. One XOR gate.
- Bit-level verification (3-bit adder, bits b2 b1 b0, subtraction done as A + B' + 1):

| Before | Event | Internal addition | C2 | C3 | Sum bits | Shown | V |
|---|---|---|---|---|---|---|---|
| 1 | +1 | 001 + 001 + 0 | 0 | 0 | 010 | 2 | **0** |
| 2 | +2 | 010 + 010 + 0 | 1 | 0 | 100 | −4 | **1** |
| 3 | +1 | 011 + 001 + 0 | 1 | 0 | 100 | −4 | **1** |
| 2 | −1 | 010 + 110 + 1 | 1 | 1 | 001 | 1 | **0** |
| −3 | −3 | 101 + 100 + 1 | 0 | 1 | 010 | 2 | **1** |
| −4 | −3 | 100 + 100 + 1 | 0 | 1 | 001 | 1 | **1** |

- V = 1 on exactly the four wrong rows.

```
          b2 stage (sign bit)
 C2 ──────►┌──────┐
           │ FA2  ├──► C3
           └──────┘
 C2 ─┐
     XOR ──► V   (overflow flag)
 C3 ─┘
```

### (c) Does the flag need changes for subtraction?
- **No.** In subtract mode the circuit adds A to B' with carry-in 1, so the adder still performs an ordinary n-bit addition.
- C2 and C3 are the same physical wires in both modes, and the same overflow rule applies to "the two numbers actually added". Rows 5 and 6 (S = 1) are flagged correctly.

**Key Logic & Takeaway**
- Bit-width detective: a correct row gives a lower bound; a wrong row gives the modulus (wrong = true ± 2^n).
- `V = C(n−1) ⊕ Cn` works unchanged for add and subtract.
- Signed overflow requires same-sign operands as **seen by the adder** (B is inverted in subtract mode).

---

## Tut-3 Q3 — Reading Circuits, Not Writing Them

### (a) The `mystery` module
- Gate primitives list the output first:
  - `and(w3, w1, r)` → `w3 = w1·r`
  - `or(f, w3, w2)` → `f = w3 + w2`
  - `and(w1, p, q)` → `w1 = pq`
  - `or(w2, q, s)` → `w2 = q + s`
- The order of the lines does not matter: Verilog structural statements are concurrent wiring.

```
 p ─┐
    AND── w1 ─┐
 q ─┤         AND── w3 ─┐
    │    r ───┘         OR── f
    └────┐              │
         OR── w2 ───────┘
 s ──────┘
```

- `f = pqr + q + s = q(pr + 1) + s = `**`q + s`** (absorption).
- p and r have **no effect**: the module is a 2-input OR of q and s.

### (b) Full subtractor built from two `half_subtractor` instances

```
                 ┌──────────────┐ d1                  ┌──────────────┐
  x ────────────►│ .x       .d  ├─────────┐  bin ────►│ .x       .d  ├──────► d
                 │     HS1      │         └──────────►│ .y   HS2     │
  y ────────────►│ .y       .b  ├── b1 ─┐             │          .b  ├── b2 ─┐
                 └──────────────┘       │             └──────────────┘       │
                                        └───────────────[OR]─────────────────┴──► bout
  HS2 as wired computes  bin − d1   (it should compute d1 − bin)
```

- **d:** HS2 gives `bin ⊕ d1 = x ⊕ y ⊕ bin` → **correct for all inputs** (XOR is symmetric).
- **bout:** HS2's borrow as wired is `bin'·d1`, but the correct second-stage borrow is `d1'·bin`. **Wrong.**
- **Failing input** `(x, y, bin) = (0, 0, 1)`:
  - d1 = 0, b1 = 0.
  - As wired: `b2 = bin'·d1 = 0·0 = 0` → bout = 0 ✗.
  - Correct: `0 − 0 − 1 = −1` needs a borrow → bout = 1.
- Full comparison:

| x y bin | correct bout | wired bout | |
|---|---|---|---|
| 0 0 0 | 0 | 0 | |
| 0 0 1 | 1 | **0** | ✗ |
| 0 1 0 | 1 | 1 | |
| 0 1 1 | 1 | 1 | |
| 1 0 0 | 0 | **1** | ✗ (d1 = 1, bin = 0 → bin'd1 = 1) |
| 1 0 1 | 0 | 0 | |
| 1 1 0 | 0 | 0 | |
| 1 1 1 | 1 | **0** | ✗ |

- **Wrong connection:** HS2's port mapping. **Fix:** `half_subtractor HS2(.x(d1), .y(bin), .d(d), .b(b2));`.

### (c) Testbench declarations

```verilog
module tb_full_subtractor;
  wire t_x, t_y, t_bin;   // ✗ assigned in initial → must be reg
  reg  t_d, t_bout;       // ✗ driven by DUT outputs → must be wire
  ...
```

- **No, it will not simulate.** All five declarations have the wrong type.
- `t_x, t_y, t_bin` are assigned procedurally inside `initial`. A `wire` is a physical connection with no storage; it only carries what a continuous driver (gate output, `assign`, module output) puts on it, and nothing stores a value "set" at time 0. A signal set procedurally must hold its value until the next assignment, which is exactly what `reg` does. → **`reg`**.
- `t_d, t_bout` are connected to the DUT's **output** ports, which drive them continuously. A `reg` can only be written procedurally; it cannot be driven by a module output. → **`wire`**.
- Corrected:

```verilog
module tb_full_subtractor;
  reg  t_x, t_y, t_bin;
  wire t_d, t_bout;
  full_subtractor DUT(.x(t_x), .y(t_y), .bin(t_bin), .d(t_d), .bout(t_bout));
  initial begin
    t_x = 0; t_y = 0; t_bin = 0;
    #5 t_x = 0; t_y = 1; t_bin = 0;
  end
endmodule
```

**Key Logic & Takeaway**
- Simplify any structural netlist before trusting it: absorption often removes inputs.
- Non-symmetric blocks (subtractors) expose port-order bugs; XOR outputs hide them.
- Testbench: what **you** drive = `reg`; what the **DUT** drives = `wire`.

---

## Tut-3 Q4 — How Do Adders Scale?

### (a) Ripple-carry adder
- Gate count: n full adders × a fixed number of gates each → **O(n)**.
- Delay: `2n + 1` gate levels → **O(n)**. Doubling n doubles the delay.

### (b) Flat carry-lookahead for n bits
- Small cases:
  - n = 1: `C1 = G0 + P0C0` → 2 product terms.
  - n = 2: C2 has 3 terms; total 2 + 3 = 5.
  - n = 4: C1…C4 have 2 + 3 + 4 + 5 = 14 terms.
  - General: Ci has i + 1 terms → total `Σ(i+1) ≈ n²/2` → **gates O(n²)**.
- **Delay O(1):** every carry is a flat 2-level AND-OR, so P/G (1) + carries (2) + sum XOR (1) ≈ 4 levels for any n.
- **Practical problem:** the last term of Cn, `P(n−1)…P0·C0`, needs an AND gate with **n + 1 inputs**. Huge fan-in is not physically realisable, and big gates are slow anyway.

### (c) 4-bit CLA blocks chained (block-ripple), n = 64
- 16 blocks; each passes its carry to the next after a constant block delay `t_blk` (a few gate levels).
- **Total ≈ (n/4) × t_blk = 16 × t_blk → still O(n).**
- Improved: the **constant factor** (slope about 4× smaller than plain ripple).
- Not changed: the **growth rate**; doubling n still doubles the delay.

### (d) Hierarchical lookahead
- Block signals from the 4 bit-level G/P:
  - `G_blk = G3 + P3G2 + P3P2G1 + P3P2P1G0` (block generates a carry by itself).
  - `P_blk = P3P2P1P0` (block passes an incoming carry straight through).
- A second-level lookahead unit (identical to (b), one level up) computes every block's carry-in directly from `G_blk`, `P_blk`.

```
  Level 0:  64 bits  → 16 groups of 4  (bit G/P → block G/P)
  Level 1:  16 blocks →  4 groups of 4 (block G/P → super-block G/P)
  Level 2:   4 super-blocks → 1 group  (→ C64 and all super-block carries)
            then carries flow back down: level 2 → level 1 → level 0 → sums
```

- **Delay:** each level adds a constant, and the number of levels is `log₄ n` → **O(log n)**.
- **Gates:** each level has 1/4 as many units as the level below (16 + 4 + 1 + …), a geometric series → **O(n)** total.
- **n = 64:** `log₄ 64 = 3` levels. About 3 × 4 ≈ **12 gate delays**, versus **129** for ripple-carry (≈ 10× faster).

**Key Logic & Takeaway**

| Design | Gates | Delay |
|---|---|---|
| Ripple | O(n) | O(n) |
| Flat CLA | O(n²) | O(1), but fan-in explodes |
| Chained 4-bit CLAs | O(n) | O(n), smaller slope |
| Hierarchical CLA | O(n) | O(log n) |

---

# TUTORIAL 4 — 4-Variable K-maps with Don't-Cares

## Tut-4 Q1 — Relationship between MSP and MPS

### (a) Map with no don't-cares

| AB \ CD | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | 0 | 0 | **1** | 0 |
| 01 | 0 | 0 | **1** | **1** |
| 11 | 0 | 0 | **1** | **1** |
| 10 | 0 | 0 | **1** | 0 |

- `F = Σm(3,6,7,11,14,15)`.
- **Grouping the 1s:**
  - `CD` = whole column 11 = {3,7,15,11} → **EPI** (only PI covering m3, m11).
  - `BC` = rows 01, 11 × columns 11, 10 = {7,6,15,14} → **EPI** (only PI covering m6, m14).
  - PIs = EPIs = {CD, BC}. **MSP: `F = CD + BC`** (4 literals).
- **Grouping the 0s:**
  - `C'` = columns 00, 01 (8 cells) → **EPI**.
  - `B'D'` = four corners {0,2,8,10} → **EPI** (only one covering m2, m10).
  - `F' = C' + B'D'` → **MPS: `F = C(B + D)`**.
- **MSP = MPS?** Yes: `C(B + D) = BC + CD`. With no don't-cares, both describe the unique function.

### (b) Map with don't-cares

| AB \ CD | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | X | **1** | **1** | X |
| 01 | 0 | X | X | 0 |
| 11 | 0 | 0 | 0 | 0 |
| 10 | **1** | 0 | 0 | **1** |

- 1s: {1, 3, 8, 10}; Xs: {0, 2, 5, 7}; 0s: {4, 6, 9, 11, 12, 13, 14, 15}.
- **Grouping the 1s:**
  - `B'D'` = corners {0,2,8,10} → **EPI** (only PI on m8, m10).
  - `A'D` = {1,3,5,7} → PI.
  - `A'B'` = {0,1,2,3} → PI.
  - m1 and m3 are each covered by both `A'D` and `A'B'` → neither is essential.
  - **Two MSPs:** `F = B'D' + A'D` **or** `F = B'D' + A'B'` (4 literals each).
- **Grouping the 0s:**
  - `B` = rows 01 and 11 = {4,5,6,7,12,13,14,15} (using X5, X7) → **EPI**.
  - `AD` = {9,11,13,15} → **EPI** (only one on m9, m11).
  - `A'D'` = {0,2,4,6} → a PI (uses X0, X2) but not needed.
  - `F' = B + AD` → **MPS: `F = B'(A' + D')` = `A'B' + B'D'`**.
- **MSP = MPS?** **Not necessarily:**
  - `B'D' + A'D` treats X5, X7 as **1** (A'D covers them), but the MPS treats them as **0** (B covers them) → **different functions**.
  - `B'D' + A'B'` happens to equal the MPS.
  - Both versions agree on every specified cell, so both are valid designs.

### (c) General conclusion
- **Without don't-cares:** MSP and MPS always describe the same function (only one function fits the truth table).
- **With don't-cares:** each minimisation picks values for the Xs independently (SOP sets some Xs to 1, POS sets some to 0), so MSP and MPS **may differ as functions**. They always agree on the specified 0s and 1s. Never assume they match.

**Key Logic & Takeaway**
- An X is a free choice made separately by each minimisation.
- Corner groups (`B'D'`) and edge wraps are easy to miss; check them first.

---

## Tut-4 Q2 — Gray-Coded Input with a Parity Check

### (a) Completed truth table
- `V = 1` if `G2G1G0Y` has an even number of 1s. `P = 1` if X ∈ {2, 3, 5, 7}. When V = 0 the downstream circuit ignores P → **P = X (don't-care)** on invalid rows.
- Minterm index = `8·G2 + 4·G1 + 2·G0 + Y`.

| X | G2G1G0 | Y | m | #1s | V | P |
|---|---|---|---|---|---|---|
| 0 | 000 | 0 | 0 | 0 | 1 | 0 |
| 0 | 000 | 1 | 1 | 1 | 0 | X |
| 1 | 001 | 0 | 2 | 1 | 0 | X |
| 1 | 001 | 1 | 3 | 2 | 1 | 0 |
| 2 | 011 | 0 | 6 | 2 | 1 | 1 |
| 2 | 011 | 1 | 7 | 3 | 0 | X |
| 3 | 010 | 0 | 4 | 1 | 0 | X |
| 3 | 010 | 1 | 5 | 2 | 1 | 1 |
| 4 | 110 | 0 | 12 | 2 | 1 | 0 |
| 4 | 110 | 1 | 13 | 3 | 0 | X |
| 5 | 111 | 0 | 14 | 3 | 0 | X |
| 5 | 111 | 1 | 15 | 4 | 1 | 1 |
| 6 | 101 | 0 | 10 | 2 | 1 | 0 |
| 6 | 101 | 1 | 11 | 3 | 0 | X |
| 7 | 100 | 0 | 8 | 1 | 0 | X |
| 7 | 100 | 1 | 9 | 2 | 1 | 1 |

- ⚠️ EXAM TRICK: the rows are listed in Gray order, but the **minterm column** comes from the binary value of G2G1G0Y. Always compute m explicitly.

### (b) V: PIs, EPIs, MSP
- `V = Σm(0,3,5,6,9,10,12,15)` (even number of 1s).

| G2G1 \ G0Y | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | **1** | 0 | **1** | 0 |
| 01 | 0 | **1** | 0 | **1** |
| 11 | **1** | 0 | **1** | 0 |
| 10 | 0 | **1** | 0 | **1** |

- Perfect checkerboard: no two 1s are adjacent (including wrap-around).
- **PIs = the 8 minterms themselves; all 8 are EPIs.**
- **MSP:** `V = G2'G1'G0'Y' + G2'G1'G0Y + G2'G1G0'Y + G2'G1G0Y' + G2G1'G0'Y + G2G1'G0Y' + G2G1G0'Y' + G2G1G0Y` (32 literals).
- Practical circuit: `V = (G2 ⊕ G1 ⊕ G0 ⊕ Y)'` (3 XORs + 1 inverter, or 2 XOR + 1 XNOR).

### (c) P with invalid rows as don't-cares
- `P = Σm(5,6,9,15) + d(1,2,4,7,8,11,13,14)`; 0s at {0, 3, 10, 12}.

| G2G1 \ G0Y | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | 0 | X | 0 | X |
| 01 | X | **1** | X | **1** |
| 11 | 0 | X | **1** | X |
| 10 | X | **1** | X | 0 |

- **PIs** (each is a group of 4 containing no 0):
  - `G0'Y` = {1,5,9,13}
  - `G1Y` = {5,7,13,15}
  - `G1G0` = {6,7,14,15}
  - `G2'G1` = {4,5,6,7}
  - `G2Y` = {9,11,13,15}
  - `G2'G0Y'` = {2,6}
  - `G2G1'G0'` = {8,9}
- **EPIs: none.** Each required 1 lies in at least two PIs (m5: G0'Y, G1Y, G2'G1; m6: G1G0, G2'G1, G2'G0Y'; m9: G0'Y, G2Y, G2G1'G0'; m15: G1Y, G1G0, G2Y). This is a **cyclic** cover.
- **Two minimal SOPs (4 literals each):**
  - **`P = G0'Y + G1G0`**
  - **`P = G2'G1 + G2Y`**
- Check `G0'Y + G1G0` on the 0s: m0 (0000) → 0 ✓; m3 (0011) → G0 = 1 kills the first term, G1 = 0 kills the second → 0 ✓; m10 (1010) → Y = 0, G1 = 0 → 0 ✓; m12 (1100) → Y = 0, G0 = 0 → 0 ✓.

### (d) Force P = 0 on invalid rows
- `P = Σm(5,6,9,15)` with no don't-cares. These four minterms are pairwise non-adjacent (they all have even parity), so nothing merges.
- **MSP:** `P = G2'G1G0'Y + G2'G1G0Y' + G2G1'G0'Y + G2G1G0Y` → **16 literals**, versus **4** in (c).
- Lesson: telling the designer "the output is ignored when V = 0" is worth a 4× reduction in logic.

**Key Logic & Takeaway**
- Parity functions are checkerboards: no K-map reduction is possible, so use XOR.
- Inputs the downstream circuit ignores are don't-cares: use them.
- No EPIs means a cyclic map with multiple equally good answers.

---

## Tut-4 Q3 — K-map Reduction Practice

### (a)

| AB \ CD | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | 0 | 0 | 0 | X |
| 01 | 0 | **1** | **1** | 0 |
| 11 | 0 | 0 | X | 0 |
| 10 | X | **1** | **1** | **1** |

- 1s {5, 7, 9, 10, 11}; Xs {2, 8, 15}.
- **PIs:** `AB'` {8,9,10,11}, `A'BD` {5,7}, `BCD` {7,15}, `ACD` {11,15}, `B'CD'` {2,10}.
- **EPIs:** `AB'` (only PI covering m9) and `A'BD` (only PI covering m5).
- All 1s are covered by the EPIs.
- **MSP: `F = AB' + A'BD`** (5 literals).

### (b)

| AB \ CD | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | 0 | X | 0 | X |
| 01 | X | 0 | **1** | 0 |
| 11 | 0 | 0 | 0 | 0 |
| 10 | **1** | **1** | **1** | **1** |

- 1s {7, 8, 9, 10, 11}; Xs {1, 2, 4}.
- **PIs:** `AB'` {8,9,10,11}, `B'C'D` {1,9}, `B'CD'` {2,10}, `A'BCD` {7}.
  - m7's neighbours m3, m5, m6, m15 are all 0, so m7 is a PI on its own.
- **EPIs:** `AB'` (m8, m11) and `A'BCD` (m7).
- **MSP: `F = AB' + A'BCD`** (6 literals).
- ⚠️ EXAM TRICK: X4 looks tempting next to m7, but they are not adjacent (0100 vs 0111 differ in 2 bits).

### (c)

| AB \ CD | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | **1** | **1** | **1** | **1** |
| 01 | 0 | X | 0 | X |
| 11 | X | **1** | 0 | 0 |
| 10 | **1** | 0 | 0 | **1** |

- 1s {0, 1, 2, 3, 8, 10, 13}; Xs {5, 6, 12}.
- **PIs:** `A'B'` {0,1,2,3}, `B'D'` {0,2,8,10}, `BC'D` {5,13}, `ABC'` {12,13}, `A'C'D` {1,5}, `A'CD'` {2,6}, `AC'D'` {8,12}.
- **EPIs:** `A'B'` (only PI on m3) and `B'D'` (only PI on m10).
- Remaining 1: m13, covered by `BC'D` or `ABC'`.
- **Two MSPs:** `F = A'B' + B'D' + BC'D` **or** `F = A'B' + B'D' + ABC'` (7 literals).

### (d)

| AB \ CD | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | X | 0 | 0 | X |
| 01 | 0 | 0 | **1** | 0 |
| 11 | **1** | 0 | 0 | 0 |
| 10 | X | **1** | **1** | **1** |

- 1s {7, 9, 10, 11, 12}; Xs {0, 2, 8}.
- **PIs:** `AB'` {8,9,10,11}, `B'D'` {0,2,8,10}, `AC'D'` {8,12}, `A'BCD` {7}.
- **EPIs:** `AB'` (m9, m11), `AC'D'` (only one on m12), `A'BCD` (m7 is isolated).
- `B'D'` is a PI but is not needed (m10 is already covered by AB').
- **MSP: `F = AB' + AC'D' + A'BCD`** (9 literals).

**Key Logic & Takeaway**
- Always look for isolated 1s first: they are their own EPIs (m7 in (b) and (d)).
- Use an X only if it enlarges a group that covers a real 1. Groups made mostly of Xs (like `B'D'` in (d)) are PIs, but you often do not need them.
- When the remaining 1s can be covered in two ways, give both MSPs.

---

# TUTORIAL 5 — Decoders, Multiplexers and Their Applications

## Tut-5 Q1 — Decoders as Minterm and Maxterm Generators

### Background in 3 bullets
- Active-high n-to-2ⁿ decoder: output `Dk = mk` (minterm k) → OR the 1-rows to get F.
- Active-low decoder: output `Dk' = mk' = Mk` (maxterm k) → AND the 0-rows (POS) **or** NAND the 1-rows (SOP).
- A single required minterm/maxterm needs **no gate at all**.

### (a) Active-high decoder

**(a)1 — 1-bit comparator with one 2-to-4 decoder + one OR**
- Decoder inputs X (MSB), Y → `D0 = X'Y'`, `D1 = X'Y`, `D2 = XY'`, `D3 = XY`.

| X | Y | X>Y | X=Y |
|---|---|---|---|
| 0 | 0 | 0 | 1 |
| 0 | 1 | 0 | 0 |
| 1 | 0 | 1 | 0 |
| 1 | 1 | 0 | 1 |

- `X>Y = m2 = D2` → **wire only, no gate**.
- `X=Y = m0 + m3 = D0 + D3` → **the one OR gate**.

```
          ┌─────────┐ D0 ───────┐
 X ──────►│A  2-to-4│ D1 (unused)│
 Y ──────►│B   DEC  │ D2 ────────┼──────────────► X>Y
          │  (act-H)│ D3 ───────┐│
          └─────────┘           OR ─────────────► X=Y
```

**(a)2 — `H(A,B,C) = A'C + AB'C' + ABC` with a 3-to-8 decoder**
- Expand to minterms:
  - `A'C` = A'B'C + A'BC = m1 + m3
  - `AB'C'` = m4
  - `ABC` = m7
- `H = Σm(1,3,4,7)` → **4-input OR of D1, D3, D4, D7**.

**(a)3 — `F(X,Y) = Σm(0,2,3)` with a 2-to-4 decoder**
- `F = D0 + D2 + D3` (3-input OR).
- Cheaper: F has a single 0 (m1), so `F = (m1)' = D1'` → **one inverter**, or a NOR with only D1 as input.
- ⚠️ EXAM TRICK: always compare #1s and #0s and use the smaller side.

**(a)4 — `G(A,B,C) = Σm(0,1,3,6)` with a 3-to-8 decoder**
- `G = D0 + D1 + D3 + D6` (4-input OR). Equally, NOR of D2, D4, D5, D7 (also 4 inputs).

```
          ┌──────────┐ D0 ──┐
 A ──────►│A  3-to-8 │ D1 ──┤
 B ──────►│B   DEC   │ D3 ──┼─OR──► G
 C ──────►│C (act-H) │ D6 ──┘
          └──────────┘ (D2,D4,D5,D7 unused)
```

### (b) Active-low decoder (outputs are maxterms `Mk`)

**(b)1 — `H(A,B,C) = B·(A+B+C')·(A'+B'+C)` with a 3-to-8 decoder**
- Turn each factor into maxterms:
  - `B` = 0 whenever B = 0 → rows 000, 001, 100, 101 → `M0 M1 M4 M5`.
  - `A + B + C'` = 0 at 001 → `M1` (already included).
  - `A' + B' + C` = 0 at 110 → `M6`.
- `H = ΠM(0,1,4,5,6) = Σm(2,3,7)`.
- Two options:
  - AND of active-low outputs 0, 1, 4, 5, 6 → **5-input AND**.
  - **NAND of active-low outputs 2, 3, 7** → 3-input NAND ✓ (cheaper).
- Why the NAND works: `NAND(M2, M3, M7) = M2' + M3' + M7' = m2 + m3 + m7`.

```
          ┌───────────┐ Y2' ──┐
 A ──────►│A  3-to-8  │ Y3' ──┼─NAND──► H = m2 + m3 + m7
 B ──────►│B  DEC     │ Y7' ──┘
 C ──────►│C (act-LOW)│
          └───────────┘
```

**(b)2 — `F(X,Y) = ΠM(1)` with a 2-to-4 decoder**
- A single maxterm: `F = M1 = X + Y'` = active-low output Y1' itself. **No gate: wire Y1' to F.**

**(b)3 — `G(A,B,C) = ΠM(2,4,5,7)` with a 3-to-8 decoder**
- **4-input AND of active-low outputs 2, 4, 5, 7.**
- Equivalent: `G = Σm(0,1,3,6)` (the same function as (a)4) → 4-input NAND of outputs 0, 1, 3, 6. Same cost.

### (c) Hierarchical: `F(W,X,Y,Z) = Σm(1,4,7,9,10)` using only 2-to-4 decoders + one OR
- Build a 4-to-16 decoder as a tree: one 2-to-4 on (W, X) drives the **enables** of 2-to-4 decoders on (Y, Z).
- The decoder for WX = 11 (minterms 12–15) is not needed → **4 decoders + 1 OR**.

| Minterm | WXYZ | Enabled by | Output used |
|---|---|---|---|
| 1 | 00 01 | E0 (W'X') | DEC-0 output 1 |
| 4 | 01 00 | E1 (W'X) | DEC-1 output 0 |
| 7 | 01 11 | E1 | DEC-1 output 3 |
| 9 | 10 01 | E2 (WX') | DEC-2 output 1 |
| 10 | 10 10 | E2 | DEC-2 output 2 |

```
                        ┌──────────┐ 0
              Y,Z ─────►│ DEC-0    │ 1 ───────────────────┐ m1
             ┌─────────►│EN        │ 2,3                  │
             │          └──────────┘                      │
 ┌────────┐  │E0        ┌──────────┐ 0 ───────────────────┤ m4
 │ 2-to-4 ├──┘ Y,Z ────►│ DEC-1    │ 1,2                  │
W│ (W,X)  ├──E1────────►│EN        │ 3 ───────────────────┤ m7     5-input
X│        ├──E2──┐      └──────────┘                      ├──OR──► F
 │        ├──E3  │      ┌──────────┐ 0                    │
 └────────┘ (NC) │ Y,Z─►│ DEC-2    │ 1 ───────────────────┤ m9
                 └─────►│EN        │ 2 ───────────────────┘ m10
                        └──────────┘ 3
```

- ⚠️ EXAM TRICK: the second-level decoders must have an enable input. The MSB pair (W, X) goes on the first level; if you swap it with (Y, Z), the minterm numbering comes out permuted.

**Key Logic & Takeaway**
- Active-high = minterms → OR (1-rows) / NOR (0-rows). Active-low = maxterms → AND (0-rows) / NAND (1-rows).
- Count the 1s and 0s and pick the gate with fewer inputs; a lone minterm or maxterm is just a wire (or inverter).

---

## Tut-5 Q2 — Implementing Expressions with Multiplexers

### (a) 3-variable functions on an 8-to-1 mux (all variables on the selects)
- Rule: `Ik` = F at row k.

| Function | Minterms | I0 | I1 | I2 | I3 | I4 | I5 | I6 | I7 |
|---|---|---|---|---|---|---|---|---|---|
| **1.** `H = (A+B+C)(A+B'+C')(A'+B+C')` = ΠM(0,3,5) | Σm(1,2,4,6,7) | 0 | 1 | 1 | 0 | 1 | 0 | 1 | 1 |
| **2.** `F = Σm(0,2,5,7)` | 0,2,5,7 | 1 | 0 | 1 | 0 | 0 | 1 | 0 | 1 |
| **3.** `G = Σm(1,3,4,5,6)` | 1,3,4,5,6 | 0 | 1 | 0 | 1 | 1 | 1 | 1 | 0 |

- For H: `(A+B+C)` = 0 at 000 → M0; `(A+B'+C')` = 0 at 011 → M3; `(A'+B+C')` = 0 at 101 → M5.

```
        ┌────────────┐
 0 ────►│I0          │
 1 ────►│I1          │
 1 ────►│I2          │
 0 ────►│I3   8:1    ├────► H
 1 ────►│I4   MUX    │
 0 ────►│I5          │
 1 ────►│I6          │
 1 ────►│I7          │
        └─┬───┬───┬──┘
          A   B   C      (S2 S1 S0)
```

### (b) 4-variable functions on an 8-to-1 mux (implementation tables)

**(b)1 — `G(A,B,C,D) = Σm(2,4,6,12,13,15)`, selects B, C, D; A on the data lines**
- ⚠️ EXAM TRICK: here the **MSB A** is the data variable, not D. Select value k = BCD; the two minterms in column k are `k` (A = 0) and `k + 8` (A = 1).

| BCD = k | 000 | 001 | 010 | 011 | 100 | 101 | 110 | 111 |
|---|---|---|---|---|---|---|---|---|
| A = 0 (m = k) | 0 | 1 | **2** | 3 | **4** | 5 | **6** | 7 |
| A = 1 (m = k+8) | 8 | 9 | 10 | 11 | **12** | **13** | 14 | **15** |
| Ik | 0 | 0 | A' | 0 | 1 | A | A' | A |

**(b)2 — `H(A,B,C,D) = Σm(1,4,11,14) + d(2,7)`, selects A, B, C; D on the data lines**
- Select k = ABC covers minterms 2k (D = 0) and 2k + 1 (D = 1).

| ABC = k | 000 | 001 | 010 | 011 | 100 | 101 | 110 | 111 |
|---|---|---|---|---|---|---|---|---|
| D = 0 | 0 | X2 | **4** | 6 | 8 | 10 | 12 | **14** |
| D = 1 | **1** | 3 | 5 | X7 | 9 | **11** | 13 | 15 |
| Ik | D | 0 | D' | 0 | 0 | D | 0 | D' |

- Xs: column 001 has X2 and 0 → choose X2 = 0 → `0` (a constant is simplest). Column 011 has 0 and X7 → `0`.
- The Xs could also give `D'` and `D`, but constants are cheaper.

**(b)3 — `F(A,B,C,D) = Σm(0,5,7,8,10,11,12,14,15)`, selects A, B, C; D on the data lines**

| ABC = k | 000 | 001 | 010 | 011 | 100 | 101 | 110 | 111 |
|---|---|---|---|---|---|---|---|---|
| D = 0 | **0** | 2 | 4 | 6 | **8** | **10** | **12** | **14** |
| D = 1 | 1 | 3 | **5** | **7** | 9 | **11** | 13 | **15** |
| Ik | D' | 0 | D | D | D' | 1 | D' | 1 |

```
         ┌────────────┐
 D' ────►│I0          │
 0  ────►│I1          │
 D  ────►│I2          │
 D  ────►│I3   8:1    ├────► F
 D' ────►│I4   MUX    │
 1  ────►│I5          │
 D' ────►│I6          │
 1  ────►│I7          │
         └─┬───┬───┬──┘
           A   B   C
```

### (c) `H(A,B,C,D) = Σm(0,2,5,6,8,9,11,15)` on a 4-to-1 mux (selects A, B)
- Each data input is a function of C, D: read the K-map row for that AB value (cells in order CD = 00, 01, 11, 10).

| AB \ CD | 00 | 01 | 11 | 10 | Ik |
|---|---|---|---|---|---|
| 00 (I0) | **1** (m0) | 0 | 0 | **1** (m2) | `D'` |
| 01 (I1) | 0 | **1** (m5) | 0 | **1** (m6) | `C ⊕ D` = `C'D + CD'` |
| 11 (I3) | 0 | 0 | **1** (m15) | 0 | `CD` |
| 10 (I2) | **1** (m8) | **1** (m9) | **1** (m11) | 0 | `C' + D` |

- ⚠️ EXAM TRICK: the K-map lists row 11 **before** row 10, but the mux input for AB = 10 is **I2** and for AB = 11 is **I3**.

```
               ┌────────────┐
 D' ──────────►│I0          │
 C⊕D ─────────►│I1   4:1    ├────► H
 C'+D ────────►│I2   MUX    │
 CD ──────────►│I3          │
               └──┬─────┬───┘
                  A     B
 (C⊕D, C'+D, CD each need a gate: XOR, OR with an inverter, AND)
```

### (d) Full-adder sum with only two 4-to-1 muxes
- `S = X ⊕ Y ⊕ Cin = (X ⊕ Y) ⊕ Cin`.
- A 4:1 mux with data `0, 1, 1, 0` **is** a 2-input XOR of its select lines (`Y = S1'S0 + S1S0'`).
- **MUX-1:** selects (X, Y), data 0,1,1,0 → `P = X ⊕ Y`.
- **MUX-2:** selects (P, Cin), data 0,1,1,0 → `S = P ⊕ Cin`.

```
        ┌──────────┐                       ┌──────────┐
 0 ────►│I0        │                0 ────►│I0        │
 1 ────►│I1  MUX-1 ├── P = X⊕Y ─┐   1 ────►│I1  MUX-2 ├────► S = X⊕Y⊕Cin
 1 ────►│I2        │            │   1 ────►│I2        │
 0 ────►│I3        │            │   0 ────►│I3        │
        └─┬─────┬──┘            │          └─┬─────┬──┘
          X     Y               └────────────┘     Cin
                                       (s1 = P, s0 = Cin)
```

- Alternative (if complements are free): one mux with selects (Y, Cin) and data `X, X', X', X` already gives S. Two muxes are needed only when X' is not available; MUX-1 can then be used as an inverter.

**Key Logic & Takeaway**
- n vars on selects → constants. n−1 on selects → {0, 1, v, v'}. n−2 on selects → small functions (K-map each row).
- Watch which variable is on the data line (MSB in (b)1!) and the AB = 10 / 11 swap between K-map rows and mux inputs.
- A 4:1 mux with data 0,1,1,0 is an XOR gate.

---

## Tut-5 Q3 — Application Questions

### Q3.1 — 3-bit Binary → Gray with one 3-to-8 decoder + one 8-to-3 encoder
- The decoder turns binary b2b1b0 into one active line k; wiring line k to encoder input g(k) makes the encoder output the Gray code.

| Binary k | b2b1b0 | Gray | Encoder input |
|---|---|---|---|
| 0 | 000 | 000 | D0 |
| 1 | 001 | 001 | D1 |
| 2 | 010 | 011 | **D3** |
| 3 | 011 | 010 | **D2** |
| 4 | 100 | 110 | **D6** |
| 5 | 101 | 111 | **D7** |
| 6 | 110 | 101 | **D5** |
| 7 | 111 | 100 | **D4** |

```
 b2 ─┐  ┌─────────┐                  ┌─────────┐
 b1 ─┼─►│ 3-to-8  │ O0 ────────────► │D0       │
 b0 ─┘  │ DECODER │ O1 ────────────► │D1  8-to-3│──► g2
        │         │ O2 ──╲   ╱─────► │D2 ENCODER│──► g1
        │         │ O3 ───╳─────────►│D3       │──► g0
        │         │ O4 ─╲ ╱╲ ╱─────► │D4       │
        │         │ O5 ──╳──╳───────►│D5       │
        │         │ O6 ─╱ ╲╱ ╲─────► │D6       │
        │         │ O7 ────────────► │D7       │
        └─────────┘                  └─────────┘
 Exact wiring: O0→D0, O1→D1, O2→D3, O3→D2, O4→D6, O5→D7, O6→D5, O7→D4
```

- Check: input 110 (6) → O6 → D5 → output 101 ✓ (gray(6) = 6 ⊕ 3 = 101).

### Q3.2 — Sharing one decoder across three functions
- `F1 = Σm(1,3,5)`, `F2 = Σm(3,5,6)`, `F3 = Σm(1,6)`.

**(a) One 3-to-8 decoder + one OR per function**

```
          ┌──────────┐
 A ──────►│ 3-to-8   │ m0  (unused)
 B ──────►│ DECODER  │ m1 ───●─────────────────────────┐
 C ──────►│(act-high)│ m2  (unused)                     │
          │          │ m3 ───┼──●──────────┐            │
          │          │ m4  (unused)        │            │
          │          │ m5 ───┼──┼──●───────┼──●         │
          │          │ m6 ───┼──┼──┼───────┼──┼──●──────┼──●
          │          │ m7  (unused)        │  │  │      │  │
          └──────────┘       │  │  │       │  │  │      │  │
                            ┌▼──▼──▼┐     ┌▼──▼──▼┐    ┌▼──▼┐
                            │  OR   │     │  OR   │    │ OR │
                            └───┬───┘     └───┬───┘    └─┬──┘
                                F1            F2         F3
                         (m1,m3,m5)     (m3,m5,m6)    (m1,m6)
```

| Decoder line | → F1 | → F2 | → F3 |
|---|---|---|---|
| m1 | ✓ | | ✓ |
| m3 | ✓ | ✓ | |
| m5 | ✓ | ✓ | |
| m6 | | ✓ | ✓ |

**(b) AND gates used vs wasted**
- Minterms used by at least one function: {1, 3, 5, 6} → **4 used**.
- Never used: {0, 2, 4, 7} → **4 wasted** (half the decoder).

**(c) A custom AND array (the PLA idea)**
- One AND gate per distinct product term actually needed: m1, m3, m5, m6 → **4 AND gates** (vs 8).
- Fan-out to the OR gates:

| AND gate | Feeds |
|---|---|
| m1 = A'B'C | F1, F3 |
| m3 = A'BC | F1, F2 |
| m5 = AB'C | F1, F2 |
| m6 = ABC' | F2, F3 |

- **Every one of the 4 AND gates feeds more than one OR gate.** This sharing is exactly how a **PLA** saves area. Minimising each function separately (e.g. `F1 = A'C + B'C`) would create new terms nobody else can share and raise the count to 6.

### Q3.3 — Driving a multiplexed 4-digit display

**(a) Wire count**
- Dedicated drivers: 4 digits × 7 segments = **28** segment wires (plus 4 decoders).
- Shared: **7** segment lines + **4** digit-enable (common) lines = **11** wires.
  - With the 2-to-4 decoder mounted at the display end, only 2 select lines need to run → 7 + 2 = 9.

**(b) Digit-enable decoder**
- A 2-bit counter steps `S1S0 = 00 → 01 → 10 → 11 → 00 …` fast (≥ ~60 Hz per digit, so the eye sees steady digits).
- A 2-to-4 decoder (= 1-to-4 demux with data = 1) turns S1S0 into exactly one active digit-enable line:
  - `EN0 = S1'S0'`, `EN1 = S1'S0`, `EN2 = S1S0'`, `EN3 = S1S0`.
- Common-cathode digits need their common line pulled **low** to light → use an **active-low** decoder (or invert the outputs).

**(c) Complete system**

```
                 ┌──────────────┐
   clock ──────► │ 2-bit counter│──── S1 S0 ─────────────────────────┐
                 └──────────────┘                                    │
                                    S1 S0                            │ S1 S0
 BCD digit 0 (4) ──►┌─────────────┐   │                              ▼
 BCD digit 1 (4) ──►│  4-bit wide │◄──┘                    ┌────────────────┐
 BCD digit 2 (4) ──►│  4-to-1 MUX │                        │ 2-to-4 decoder │
 BCD digit 3 (4) ──►│ (quad mux)  │                        │ (digit enable) │
                    └──────┬──────┘                        └─┬───┬───┬───┬──┘
                           │ 4 bits (selected digit)         │   │   │   │
                    ┌──────▼──────┐                        EN0 EN1 EN2 EN3
                    │ BCD→7-seg   │                          │   │   │   │
                    │ decoder     │                          ▼   ▼   ▼   ▼
                    └──────┬──────┘                        ┌───┐┌───┐┌───┐┌───┐
                           │ a b c d e f g (7 shared lines)│ 0 ││ 1 ││ 2 ││ 3 │
                           └──────────────────────────────►│   ││   ││   ││   │
                                                           └───┘└───┘└───┘└───┘
                                       (segment lines go to ALL four digits in parallel)
```

**(d) Why the mux and the decoder must see the same 2-bit value**
- At each instant the segment lines carry the pattern of the digit selected by the **mux**, and it lights on whichever digit the **decoder** enables.
- If they differ (say the mux selects digit 2's value while the decoder enables digit 1), digit 1 shows digit 2's number.
- Even a slight skew at each switching edge briefly shows the next or previous digit's pattern on the wrong position → **ghosting**: faint wrong segments on neighbouring digits.
- Hence both blocks are driven by the **same wires** from the same counter. Real designs also blank the display briefly during the switch.

**Key Logic & Takeaway**
- Decoder → permuted wiring → encoder = any one-to-one code converter.
- A decoder wastes AND gates for unused minterms; a shared AND array (PLA) builds only the terms needed and shares them.
- Time-multiplexing = mux (data) + demux/decoder (destination) driven by the same select lines.

---


# TUTORIAL 6 — Latches, Flip-Flops and Sequential Circuit Analysis

## Tut-6 Q1 — Complete the Timing Diagrams

### (a) Basic SR latch (NOR-based), Q starts at 0
- NOR SR latch rules: `SR = 00` hold, `10` set (Q = 1), `01` reset (Q = 0), `11` forbidden (both outputs 0).
- From the figure: S is high in intervals 2 and 6; R is high in interval 4.

| Interval | 1 | 2 | 3 | 4 | 5 | 6 | 7 |
|---|---|---|---|---|---|---|---|
| S | 0 | 1 | 0 | 0 | 0 | 1 | 0 |
| R | 0 | 0 | 0 | 1 | 0 | 0 | 0 |
| **Q** | **0** | **1** | **1** | **0** | **0** | **1** | **1** |
| Operation | hold | **set** | hold | **reset** | hold | **set** | hold |

```
 interval:   1     2     3     4     5     6     7   
 S       : ______|‾‾‾‾‾|_________________|‾‾‾‾‾|_____
 R       : __________________|‾‾‾‾‾|_________________
 Q       : ______|‾‾‾‾‾‾‾‾‾‾‾|___________|‾‾‾‾‾‾‾‾‾‾‾
 op      :  hold  SET   hold RESET  hold  SET   hold 
```

- ⚠️ EXAM TRICK: when S returns to 0 (interval 3) Q does **not** fall: the latch remembers. Only R resets it.

### (b) Positive-edge D flip-flop with asynchronous Reset, Q starts at 0
- Reading the figure (one time unit = half a clock period):
  - Rising clock edges at **t1, t3, t5, t7, t9**.
  - D = 1 until t4, D = 0 from t4 to t6, D = 1 after t6.
  - Reset is high from **t4.5 to t5.5**.

| Event | D at that moment | Reset | Q after |
|---|---|---|---|
| start | 1 | 0 | 0 |
| ↑ t1 | 1 | 0 | **1** |
| ↑ t3 | 1 | 0 | 1 |
| t4 (D falls) | 0 | 0 | 1 (no edge → no change) |
| **t4.5 Reset ↑** | 0 | 1 | **0 immediately** (asynchronous) |
| **↑ t5 (inside reset)** | 0 | 1 | **0** (reset overrides the clock) |
| t5.5 Reset ↓ | 0 | 0 | 0 (releasing reset does **not** load D; it waits for an edge) |
| ↑ t7 | 1 | 0 | **1** |
| ↑ t9 | 1 | 0 | 1 |

```
 time : t0    t1    t2    t3    t4    t5    t6    t7    t8    t9    
 Clk  : ______|‾‾‾‾‾|_____|‾‾‾‾‾|_____|‾‾‾‾‾|_____|‾‾‾‾‾|_____|‾‾‾‾‾
 D    : ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|___________|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
 Reset: ___________________________|‾‾‾‾‾|__________________________
 Q    : ______|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|______________|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾
        Q↑ at t1 (edge, D=1); Q↓ at t4.5 (async reset); edge t5 ignored; Q↑ at t7 (edge, D=1)
```

- ⚠️ EXAM TRICK: (1) an asynchronous reset acts **immediately**, not at the next edge; (2) a clock edge that arrives while reset is asserted is **ignored**; (3) when reset is released, Q stays 0 until the **next active edge** samples D.

**Key Logic & Takeaway**
- Latch = level-sensitive memory (hold/set/reset by input levels). Flip-flop = samples D only at the active edge.
- Direct (async) inputs override everything, including the clock.

---

## Tut-6 Q2 — Gated SR Latch: Propagation Delay and Settling Time

### The circuit (every gate delay = Δ)

```
 S ──┐                        ┌───────────┐
     NAND1 ── S' ────────────►│           │
 C ──┤                        │  NAND3    ├──●──────► Q
     │                  ┌────►│           │  │
     │                  │     └───────────┘  │
     │                  │ Q'                 │ Q
     │                  │     ┌───────────┐  │
     │                  └─────┤  NAND4    │◄─┘
 C ──┤                        │           ├──●──────► Q'
     NAND2 ── R' ────────────►│           │
 R ──┘                        └───────────┘

 S' = (S·C)'   R' = (R·C)'   Q = (S'·Q')'   Q' = (R'·Q)'
```

### Timing with S = 1, R = 0, Q = 0, Q' = 1 initially; C rises at 3Δ and falls at 10Δ

| Time | Event (cause → effect) |
|---|---|
| 3Δ | C rises |
| **4Δ** | NAND1: S·C = 1 → **S' falls to 0** (1 gate delay) |
| **5Δ** | NAND3: S' = 0 → **Q rises to 1** ← Q *begins to change* |
| **6Δ** | NAND4: R' = 1, Q = 1 → **Q' falls to 0** ← circuit *fully stable* |
| 7Δ | NAND3 now sees S' = 0, Q' = 0 → Q stays 1 (no further change) |
| 10Δ | C falls |
| 11Δ | S' returns to 1 |
| 12Δ | NAND3 sees S' = 1, Q' = 0 → Q = 1: **no change** |

```
 time(Δ): 0  1  2  3  4  5  6  7  8  9  10 11 12 13 
 C      : _________|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|___________
 S'     : ‾‾‾‾‾‾‾‾‾‾‾‾|____________________|‾‾‾‾‾‾‾‾  (falls 4Δ, rises 11Δ)
 Q      : _______________|‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾  (rises 5Δ)
 Q'     : ‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾‾|_______________________  (falls 6Δ)
```

- **Q first begins to change at 5Δ** (2Δ after C rises). **Fully stable at 6Δ** (3Δ after C rises), once the Q' feedback has arrived.
- **When C falls at 10Δ, Q does not change.** S' and R' both return to 1, which is the "hold" input for the cross-coupled NAND pair. Since Q' = 0 is already fed back, NAND3 keeps Q = 1. The latch stores the value.
- (Verified with a gate-level delay simulation.)

**Key Logic & Takeaway**
- Count gate levels along the path: C → input NAND (Δ) → output NAND (Δ) → other output NAND (Δ).
- A latch is "settled" only when **both** outputs have changed and the feedback loop agrees.

---

## Tut-6 Q3 — Using Latches to Manipulate Signals

### (a) Minimum C pulse width w
- When C goes high, S' falls Δ later; Q rises Δ after that; Q' falls Δ after that.
- The latch holds by itself only once **Q' = 0** has fed back into NAND3. Until then, Q is 1 only because S' is still 0.
- S' stays low until Δ after C falls (i.e. until w + Δ). It must still be low when Q' arrives at 3Δ:
  - `w + Δ ≥ 3Δ` → **`w_min = 2Δ`** (the time for the change to travel once around the feedback loop).
- **If w < 2Δ:** S' returns to 1 before Q' = 0 has arrived, so NAND3 sees (S' = 1, Q' = 1) and Q falls back to 0. Meanwhile the Q' = 0 pulse arrives and pushes Q back up. In the equal-delay model, Q and Q' then **oscillate** indefinitely (simulated: w = 0.5Δ, 1Δ, 1.5Δ all oscillate; w = 2Δ latches cleanly). In real hardware this is **metastability**: Q may hover between levels and finally settle to 0 (old value) or 1 at random.
- So a too-short pulse means Q does not reliably hold the new value.

```
 w = 2Δ (OK)                            w = 1Δ (too short)
 C   __|‾‾‾‾|___________                 C   __|‾‾|_______________
 S'  ‾‾‾|____|‾‾‾‾‾‾‾‾‾                  S'  ‾‾‾|__|‾‾‾‾‾‾‾‾‾‾‾‾‾‾
 Q   ______|‾‾‾‾‾‾‾‾‾‾‾‾ (holds 1)       Q   _____|‾|_|‾|_|‾|_|‾|_   (oscillates)
 Q'  ‾‾‾‾‾‾‾‾|__________                 Q'  ‾‾‾‾‾‾‾|_|‾|_|‾|_|‾|_
```

### (b) Two latches in series sharing C (intended 1-bit shift per pulse)

```
            ┌─────────┐          ┌─────────┐
 Data_In ──►│D       Q├── Q1 ───►│D       Q├──► Output
            │ Latch 1 │          │ Latch 2 │
       ┌───►│C      Q'│     ┌───►│C      Q'│
       │    └─────────┘     │    └─────────┘
 C ────┴────────────────────┘
```

- **Yes, there is a maximum safe width.**
- While C = 1, **both** latches are transparent. Latch 1's Q1 becomes Data_In one propagation delay `t_pd` (≈ 2–3Δ for these latches) after C rises.
- If C is still high when the new Q1 arrives at Latch 2's input, Latch 2 (also transparent) passes it straight through → after the pulse **both latches hold Data_In**, and the old value of Latch 1 is lost. This is **race-through**: the data moved two stages in one pulse.
- **Condition:** `w < t_pd(Latch 1, C→Q)`.
- **The real problem:** the minimum width from (a) is ≈ 2Δ and the maximum is also ≈ 2Δ (one latch delay). With identical latches the valid window is about zero width, so the circuit cannot be made reliable over temperature and process variation.

### (c) How edge-triggered flip-flops remove the problem
- An edge-triggered (e.g. master–slave) flip-flop samples D only in a tiny window around the clock edge (setup + hold time) and changes Q only **after** the edge.
- **Internally:** for a positive-edge FF the master latch is transparent while C = 0 and the slave while C = 1. They are **never transparent at the same time**, so data cannot race through.
- In a chain, FF2 samples Q1 at the same edge that FF1 samples Data_In. Q1 only changes `t_pd` after the edge, which is longer than FF2's hold time, so FF2 captures the **old** Q1. Exactly one shift per edge.
- Pulse width no longer matters (beyond a minimum high/low time); there is **no maximum**. Timing reduces to setup/hold checks at one instant.

**Key Logic & Takeaway**
- Latches have both a minimum pulse width (loop must close: 2Δ) and a maximum (no race-through: < t_pd). For identical latches these coincide → unreliable.
- Edge triggering = master + slave latches on opposite clock phases → only the edge matters.

---

## Tut-6 Q4 — Design a JK Flip-Flop Using a T Flip-Flop
- A T flip-flop toggles when T = 1. So **T = 1 exactly when the JK flip-flop must change state**: `T = Q ⊕ Q+`.

| J | K | Q | Q+ (JK behaviour) | T = Q ⊕ Q+ |
|---|---|---|---|---|
| 0 | 0 | 0 | 0 (hold) | 0 |
| 0 | 0 | 1 | 1 (hold) | 0 |
| 0 | 1 | 0 | 0 (reset) | 0 |
| 0 | 1 | 1 | 0 (reset) | **1** |
| 1 | 0 | 0 | 1 (set) | **1** |
| 1 | 0 | 1 | 1 (set) | 0 |
| 1 | 1 | 0 | 1 (toggle) | **1** |
| 1 | 1 | 1 | 0 (toggle) | **1** |

| J \ K Q | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 0 | 0 | 0 | **1** | 0 |
| 1 | **1** | 0 | **1** | **1** |

- Groups: `JQ'` = {m4, m6} (J = 1, Q = 0 column pair 00 and 10 wrap) and `KQ` = {m3, m7}. Both are EPIs.
- **`T = JQ' + KQ`**.
- Sanity check: Q = 0 → T = J (set or toggle from 0 needs a toggle); Q = 1 → T = K (reset or toggle from 1 needs a toggle).

```
 J ───────────┐
              AND ──┐
 Q' ──────────┘     │          ┌─────────┐
                    OR ── T ──►│T       Q├──●──► Q
 K ───────────┐     │          │  T-FF    │  │
              AND ──┘    clk ─►│>       Q'├──┼──► Q'
 Q ───────────┘                └─────────┘  │
   (Q and Q' fed back from the FF outputs) ◄┘
```

**Key Logic & Takeaway**
- Converting flip-flop X into Y: write Y's characteristic table, then use **X's excitation table** to find X's inputs, then a K-map.
- For a T target: T = Q ⊕ Q+.

---

## Tut-6 Q5 — A New Flip-Flop Type (A, B): D-based vs JK-based

| A | B | Q+ |
|---|---|---|
| 0 | 0 | 0 (reset) |
| 0 | 1 | Q' (toggle) |
| 1 | 0 | Q' (toggle) |
| 1 | 1 | 1 (set) |

### Expanded table with both implementations

| A | B | Q | Q+ | D | J | K |
|---|---|---|---|---|---|---|
| 0 | 0 | 0 | 0 | 0 | 0 | X |
| 0 | 0 | 1 | 0 | 0 | X | 1 |
| 0 | 1 | 0 | 1 | 1 | 1 | X |
| 0 | 1 | 1 | 0 | 0 | X | 1 |
| 1 | 0 | 0 | 1 | 1 | 1 | X |
| 1 | 0 | 1 | 0 | 0 | X | 1 |
| 1 | 1 | 0 | 1 | 1 | 1 | X |
| 1 | 1 | 1 | 1 | 1 | X | 0 |

### (a) Using a D flip-flop: D = Q+
- `D = Σm(2,4,6,7)` over (A, B, Q):

| A \ B Q | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 0 | 0 | 0 | 0 | **1** |
| 1 | **1** | 0 | **1** | **1** |

- PIs (all essential): `BQ'` {2,6}, `AQ'` {4,6}, `AB` {6,7}.
- **`D = AQ' + BQ' + AB`** → **6 literals** (3 two-input ANDs + a 3-input OR).

### (b) Using a JK flip-flop
- **J** (only rows with Q = 0 matter; Q = 1 rows are X):

| A \ B Q | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 0 | 0 | X | X | **1** |
| 1 | **1** | X | X | **1** |

  - **`J = A + B`** (2 literals).
- **K** (only rows with Q = 1 matter; Q = 0 rows are X):

| A \ B Q | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 0 | X | **1** | **1** | X |
| 1 | X | **1** | 0 | X |

  - K = 0 only at A = B = 1 → **`K = A' + B' = (AB)'`** (2 literals; a single NAND gate).
- **Total: 4 literals** (one OR + one NAND).

```
 D version                         JK version
 A ─┬─AND(A,Q')─┐                  A ─┬─────OR────── J ─┐  ┌──────┐
 B ─┼─AND(B,Q')─OR── D ─► D-FF     B ─┼──┘                 └─►│J    │
    └─AND(A,B)──┘                     └─────NAND──── K ──────►│K  Q ├─► Q
                                                              └──────┘
```

### Comparison
- **D: 6 literals vs JK: 4 literals → the JK version is cheaper.**
- **Why:** every row of the JK excitation table has one don't-care (when Q = 0, K is free; when Q = 1, J is free). Half the cells in each of the J and K maps are X, giving large groups. The D excitation table has **no** don't-cares (D must equal Q+ exactly), so its map is fully specified.

**Key Logic & Takeaway**
- JK maps: half X's → small equations. D maps: no X's → simple to derive, usually more logic.
- Read J from the Q = 0 half and K from the Q = 1 half.

---

## Tut-6 Q6 — Analyse This Sequential Circuit
- Given: two T flip-flops (Q1 Q0), input X, output Z:
  - `T1 = X·Q0`
  - `T0 = X`
  - `Z = X·Q1·Q0` (depends on the input → **Mealy** output)

```
            ┌──────────────────────────┐
 X ───●─────┼──────────► T0 ┌─────┐    │
      │     │               │ FF0 ├─Q0─●──────┐
      │     │         clk ─►│ T   │    │      │
      │     └─AND(X,Q0)──► T1 ┌─────┐ │      │
      │                       │ FF1 ├─Q1───┐  │
      │                 clk ─►│ T   │      │  │
      │                       └─────┘      │  │
      └──────────────── AND(X, Q1, Q0) ◄───┴──┘ ──► Z
```

### (a) State table
- Procedure: compute T1, T0 from the present state and input; apply `Q+ = Q ⊕ T` to each flip-flop.

| Q1 Q0 | X | T1 = XQ0 | T0 = X | Q1+ Q0+ | Z = XQ1Q0 |
|---|---|---|---|---|---|
| 0 0 | 0 | 0 | 0 | 0 0 | 0 |
| 0 0 | 1 | 0 | 1 | 0 1 | 0 |
| 0 1 | 0 | 0 | 0 | 0 1 | 0 |
| 0 1 | 1 | 1 | 1 | 1 0 | 0 |
| 1 0 | 0 | 0 | 0 | 1 0 | 0 |
| 1 0 | 1 | 0 | 1 | 1 1 | 0 |
| 1 1 | 0 | 0 | 0 | 1 1 | 0 |
| 1 1 | 1 | 1 | 1 | 0 0 | **1** |

- Compact form (next state / output):

| Present | X = 0 | X = 1 |
|---|---|---|
| 00 | 00 / 0 | 01 / 0 |
| 01 | 01 / 0 | 10 / 0 |
| 10 | 10 / 0 | 11 / 0 |
| 11 | 11 / 0 | 00 / **1** |

### (b) State diagram (Mealy: arcs labelled X/Z)

```
        0/0              0/0              0/0              0/0
       ┌───┐            ┌───┐            ┌───┐            ┌───┐
       │   ▼            │   ▼            │   ▼            │   ▼
     ┌──────┐  1/0   ┌──────┐  1/0   ┌──────┐  1/0   ┌──────┐
     │  00  ├───────►│  01  ├───────►│  10  ├───────►│  11  │
     └──────┘        └──────┘        └──────┘        └──┬───┘
        ▲                                               │
        └───────────────────── 1/1 ─────────────────────┘
```

### (c) Plain-language description
- A **2-bit (mod-4) up-counter with enable X**: it holds when X = 0 and counts up by one on each clock edge where X = 1.
- **Z = 1** for the cycle in which the counter is at 3 and X = 1 (it is about to wrap 3 → 0). So Z marks every **4th** cycle in which X = 1, acting as a carry-out / divide-by-4 of the X events.

**Key Logic & Takeaway**
- Analysis recipe: FF input equations → FF input values per row → next state via the characteristic equation (`Q+ = T ⊕ Q`) → state table → diagram → describe.
- An output that depends on X is Mealy (shown on the arcs); one that depends only on the state is Moore (shown inside the circles).
- `T0 = X`, `T1 = XQ0` is the standard synchronous binary counter pattern (`Ti = EN · Q0 … Q(i−1)`, Lec-22).

---


# TUTORIAL 7 — Sequential Design and Counters (register questions excluded)

> Tut-7 **Q4** (bottle line: a 5-stage shift-register delay) and **Q5** (serial adder/subtractor built on shift registers) are **register** questions and are skipped as you asked. Q1–Q3 are below.

## Tut-7 Q1 — Binary Counters: Down, then Up-Down

### (a) 4-bit ripple (asynchronous) DOWN counter

- **How the up counter works:** every FF has J = K = 1 (always toggles). FF0 is clocked by CLK; FF(i) is clocked by **Q(i−1)** on its **negative** edge. Bit i must toggle exactly when bit (i−1) goes **1 → 0** (a carry: `0111 → 1000`).
- **What changes for counting down:** bit i must toggle exactly when bit (i−1) goes **0 → 1** (a borrow: `1000 → 0111` means Q0 0→1, then Q1 0→1, then Q2 0→1, then Q3 1→0).
  - A 0 → 1 change on Q(i−1) is a 1 → 0 change on **Q'(i−1)**.
  - **So clock each stage from the previous stage's Q' instead of Q** (keeping negative-edge FFs). Equivalently, keep Q but use positive-edge-triggered FFs.
- Initialise with the **direct Set (preset)** inputs → 1111.

```
            ┌──────────┐       ┌──────────┐       ┌──────────┐       ┌──────────┐
  1 ──────► │J   FF0  Q├─► Q0  │J   FF1  Q├─► Q1  │J   FF2  Q├─► Q2  │J   FF3  Q├─► Q3
            │          │  1 ──►│          │  1 ──►│          │  1 ──►│          │
  1 ──────► │K       Q'├──┐    │K       Q'├──┐    │K       Q'├──┐    │K       Q'│
 CLK ─────o►│C  S      │  └──o►│C  S      │  └──o►│C  S      │  └──o►│C  S      │
            └───▲──────┘       └───▲──────┘       └───▲──────┘       └───▲──────┘
 SET' ──────────┴──────────────────┴──────────────────┴──────────────────┘
       (direct Set → start at 1111;  o► = negative-edge clock input;
        each stage clocked by the PREVIOUS stage's Q')
```

- **Trace** (first few clocks):

| CLK edge | Q0 | Q0' edge | Q1 | Q1' edge | Q2 | Q3 | Count |
|---|---|---|---|---|---|---|---|
| (preset) | 1 | — | 1 | — | 1 | 1 | 15 |
| 1 | 1→0 | 0→1 (rising, ignored) | 1 | — | 1 | 1 | 14 |
| 2 | 0→1 | 1→0 (falling) → FF1 toggles | 1→0 | 0→1 (ignored) | 1 | 1 | 13 |
| 3 | 1→0 | ignored | 0 | — | 1 | 1 | 12 |
| 4 | 0→1 | falling → FF1 toggles | 0→1 | falling → FF2 toggles | 1→0 | 1 | 11 |

- Continues 10, 9, …, 1, 0, then `0000 → 1111` (every stage borrows) and repeats.
- ⚠️ EXAM TRICK: summary table for ripple counters:

| FF triggering | Clock next stage from Q | Clock next stage from Q' |
|---|---|---|
| Negative edge | **UP** counter | **DOWN** counter |
| Positive edge | **DOWN** counter | **UP** counter |

### (b) 3-bit synchronous UP/DOWN counter (UD = 0 up, UD = 1 down)

**Using D flip-flops (D = next state)** — K-maps use rows `UD Q2`, columns `Q1 Q0`:

- **D0:** the LSB always toggles.

| UD Q2 \ Q1 Q0 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | 1 | 0 | 0 | 1 |
| 01 | 1 | 0 | 0 | 1 |
| 11 | 1 | 0 | 0 | 1 |
| 10 | 1 | 0 | 0 | 1 |

  - **`D0 = Q0'`** (1 literal).
- **D1:**

| UD Q2 \ Q1 Q0 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | 0 | 1 | 0 | 1 |
| 01 | 0 | 1 | 0 | 1 |
| 11 | 1 | 0 | 1 | 0 |
| 10 | 1 | 0 | 1 | 0 |

  - Checkerboard in UD, Q1, Q0 → no merging beyond Q2: **`D1 = UD'Q1'Q0 + UD'Q1Q0' + UDQ1'Q0' + UDQ1Q0`** (12 literals) `= Q1 ⊕ Q0 ⊕ UD`.
- **D2:** `D2 = Σm(3,4,5,6,8,13,14,15)`

| UD Q2 \ Q1 Q0 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | 0 | 0 | **1** | 0 |
| 01 | **1** | **1** | 0 | **1** |
| 11 | 0 | **1** | **1** | **1** |
| 10 | **1** | 0 | 0 | 0 |

  - EPIs: `UD'Q2'Q1Q0` (m3, isolated), `UDQ2'Q1'Q0'` (m8, isolated).
  - **`D2 = UD'Q2'Q1Q0 + UDQ2'Q1'Q0' + Q2Q1'Q0 + UD'Q2Q0' + UDQ2Q1`** (17 literals; an equally minimal alternative is `… + Q2Q1Q0' + UD'Q2Q1' + UDQ2Q0`).
- **D total = 1 + 12 + 17 = 30 literals.**

**Using T flip-flops (T = 1 when the bit changes)**

- **T0 = 1** (always toggles) → 0 literals.
- **T1:** up: Q1 toggles when Q0 = 1; down: Q1 toggles when Q0 = 0.

| UD Q2 \ Q1 Q0 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | 0 | 1 | 1 | 0 |
| 01 | 0 | 1 | 1 | 0 |
| 11 | 1 | 0 | 0 | 1 |
| 10 | 1 | 0 | 0 | 1 |

  - **`T1 = UD'Q0 + UDQ0'`** `= UD ⊕ Q0` (4 literals).
- **T2:** up: toggles when Q1Q0 = 11; down: toggles when Q1Q0 = 00.

| UD Q2 \ Q1 Q0 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | 0 | 0 | 1 | 0 |
| 01 | 0 | 0 | 1 | 0 |
| 11 | 1 | 0 | 0 | 0 |
| 10 | 1 | 0 | 0 | 0 |

  - **`T2 = UD'Q1Q0 + UDQ1'Q0'`** (6 literals).
- **T total = 0 + 4 + 6 = 10 literals.**

**Comparison**
- **T version: 10 literals vs D version: 30 literals → T is much simpler.**
- **Why:** a counter's job is to **toggle** bits in a pattern. T = 1 encodes "this bit changes", which depends only on the lower bits and UD. D must encode the **new value**, which also depends on the bit's own current value (D = Q ⊕ T), so every D map is the T map XORed with Q: no don't-cares and checkerboard-like maps. JK/T excitation tables suit counters; D's table has no freedom.

**Circuit (T-based, the simpler one)**

```
 Gate equations
   T0 = 1
   T1 = UD'·Q0 + UD·Q0'            (= UD ⊕ Q0 : one XOR also works)
   T2 = UD'·Q1·Q0 + UD·Q1'·Q0'

            ┌──────┐            ┌──────┐            ┌──────┐
  1 ──T0───►│T  Q0 ├──●──► Q0   │T  Q1 ├──●──► Q1   │T  Q2 ├──► Q2
            │  FF0 │  │    ┌───►│  FF1 │  │    ┌───►│  FF2 │
  CLK ─────►│>  Q0'├──┼─●  │    │>  Q1'├──┼─●  │    │>     │
            └──────┘  │ │  │    └──────┘  │ │  │    └──────┘
                      │ │  │              │ │  │
   UD' ── AND(UD',Q0)─┘ │  │  T1          │ │  │  T2
   UD  ── AND(UD,Q0')───┘  │              │ │  │
              └──── OR ────┘              │ │  │
   UD' ── AND(UD',Q1,Q0) ─────────────────┘ │  │
   UD  ── AND(UD,Q1',Q0') ──────────────────┘  │
              └────────────── OR ──────────────┘
   (Q0 also feeds the 3-input ANDs; one inverter makes UD'; all FFs share CLK)
```

**Key Logic & Takeaway**
- Ripple counters: the edge type and the Q/Q' choice together decide up vs down.
- Synchronous counters: `T(i) = 1` when all lower bits are 1 (up) or all lower bits are 0 (down). T/JK flip-flops give far less logic than D.

---

## Tut-7 Q2 — Counters with Unused States (mod-6, JK flip-flops)

### State diagram

```
   ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐    ┌─────┐
   │ 000 ├───►│ 001 ├───►│ 010 ├───►│ 011 ├───►│ 100 ├───►│ 101 │
   └──▲──┘    └─────┘    └─────┘    └─────┘    └─────┘    └──┬──┘
      └──────────────────────────────────────────────────────┘
   unused:  110   111   (behaviour decided by design (a) or (b))
```

### Excitation table (JK: 0→0 = 0X, 0→1 = 1X, 1→0 = X1, 1→1 = X0)

| Q2 Q1 Q0 | Next (a) | J2 K2 | J1 K1 | J0 K0 | Next (b) | J2 K2 | J1 K1 | J0 K0 |
|---|---|---|---|---|---|---|---|---|
| 000 | 001 | 0 X | 0 X | 1 X | 001 | 0 X | 0 X | 1 X |
| 001 | 010 | 0 X | 1 X | X 1 | 010 | 0 X | 1 X | X 1 |
| 010 | 011 | 0 X | X 0 | 1 X | 011 | 0 X | X 0 | 1 X |
| 011 | 100 | 1 X | X 1 | X 1 | 100 | 1 X | X 1 | X 1 |
| 100 | 101 | X 0 | 0 X | 1 X | 101 | X 0 | 0 X | 1 X |
| 101 | 000 | X 1 | 0 X | X 1 | 000 | X 1 | 0 X | X 1 |
| 110 | **XXX** | X X | X X | X X | **000** | X 1 | X 1 | 0 X |
| 111 | **XXX** | X X | X X | X X | **000** | X 1 | X 1 | X 1 |

### (a) Unused states as don't-cares — K-maps (rows Q2, columns Q1Q0)

| J2 | 00 | 01 | 11 | 10 | | K2 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| Q2=0 | 0 | 0 | **1** | 0 | | Q2=0 | X | X | X | X |
| Q2=1 | X | X | X | X | | Q2=1 | 0 | **1** | X | X |

- **`J2 = Q1Q0`**, **`K2 = Q0`**.

| J1 | 00 | 01 | 11 | 10 | | K1 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| Q2=0 | 0 | **1** | X | X | | Q2=0 | X | X | **1** | 0 |
| Q2=1 | 0 | 0 | X | X | | Q2=1 | X | X | X | X |

- **`J1 = Q2'Q0`** (cannot be just Q0: m5 = 0), **`K1 = Q0`**.
- **`J0 = 1`, `K0 = 1`** (Q0 toggles every clock; all specified cells are 1 or X).
- **Cost: 2 two-input AND gates (6 literals).**

### (b) Unused states forced to 000

| J2 | 00 | 01 | 11 | 10 | | K2 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| Q2=0 | 0 | 0 | **1** | 0 | | Q2=0 | X | X | X | X |
| Q2=1 | X | X | X | X | | Q2=1 | 0 | **1** | **1** | **1** |

- **`J2 = Q1Q0`**, **`K2 = Q1 + Q0`**.

| J1 | 00 | 01 | 11 | 10 | | K1 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| Q2=0 | 0 | **1** | X | X | | Q2=0 | X | X | **1** | 0 |
| Q2=1 | 0 | 0 | X | X | | Q2=1 | X | X | **1** | **1** |

- **`J1 = Q2'Q0`**, **`K1 = Q2 + Q0`**.

| J0 | 00 | 01 | 11 | 10 | | K0 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| Q2=0 | **1** | X | X | **1** | | Q2=0 | X | **1** | **1** | X |
| Q2=1 | **1** | X | X | 0 | | Q2=1 | X | **1** | **1** | X |

- **`J0 = Q2' + Q1'` = `(Q2Q1)'`** (one NAND), **`K0 = 1`**.
- **Cost: 2 ANDs + 2 ORs + 1 NAND (10 literals).**

```
 Design (a)                                Design (b)
 J2 = Q1·Q0     K2 = Q0                    J2 = Q1·Q0     K2 = Q1 + Q0
 J1 = Q2'·Q0    K1 = Q0                    J1 = Q2'·Q0    K1 = Q2 + Q0
 J0 = 1         K0 = 1                     J0 = (Q2·Q1)'  K0 = 1

        ┌──────┐      ┌──────┐      ┌──────┐
 J0 ───►│J  Q0 │ J1 ─►│J  Q1 │ J2 ─►│J  Q2 │      all three FFs share CLK;
 K0 ───►│K FF0 │ K1 ─►│K FF1 │ K2 ─►│K FF2 │      J/K come from the gates above,
 CLK ──►│>     │ CLK─►│>     │ CLK─►│>     │      fed by Q0, Q1, Q2 and Q2'
        └──────┘      └──────┘      └──────┘
```

### (c) What happens from 110 or 111?

**Design (a) (don't-cares)**

| From | J2 K2 (Q1Q0, Q0) | J1 K1 (Q2'Q0, Q0) | J0 K0 | Next |
|---|---|---|---|---|
| 110 | 0, 0 → hold 1 | 0, 0 → hold 1 | 1, 1 → toggle 0→1 | **111** |
| 111 | 1, 1 → toggle → 0 | 0, 1 → reset → 0 | 1, 1 → toggle → 0 | **000** ✓ |

- 110 → 111 → 000: **2 clocks** to recover. 111 → 000: **1 clock**.
- It **does** recover (it is self-starting, by luck), but it passes through an invalid count (7) on the way.

**Design (b) (explicit)**
- 110 → 000 and 111 → 000: **1 clock** each, guaranteed by design.

```
 (a)  110 ──► 111 ──► 000 ──► 001 ...          (b)  110 ──► 000 ──► 001 ...
                                                     111 ──┘
```

**Trade-off**
- Don't-care design: **cheaper** (6 vs 10 literals; no OR gates), but behaviour from the unused states is **whatever the minimised logic happens to do**. It must be checked by tracing. Here it recovers, but it could have locked into an unused loop (e.g. 110 ↔ 111 forever) in another design.
- Explicit design: **costs more gates**, but recovery is guaranteed in one cycle.
- ⚠️ EXAM TRICK: whenever you use unused states as don't-cares, **always** trace each unused state through your final equations and draw its arrows on the state diagram. "Self-starting" is something you show, not assume.

**Key Logic & Takeaway**
- Don't-cares minimise logic; explicit next states buy robustness.
- JK: read J from the Q = 0 half and K from the Q = 1 half of each map.

---

## Tut-7 Q3 — Coin-Operated Turnstile

### (a) State diagram, number of states, number of FFs
- Track the money accumulated so far: **S0 = ₹0, S5 = ₹5, S10 = ₹10**. Reaching ≥ ₹15 opens the gate for one cycle and returns to S0.
- **Mealy design:** Open is produced on the transition (it depends on the coin input). 3 states → **⌈log₂3⌉ = 2 flip-flops** (one code left unused).
- Inputs: F (₹5), T (₹10), never both 1. Arc label `FT/Open`.

```
                         00/0
                        ┌────┐
                        │    ▼
                   ┌────────────┐
      ┌───────────►│  S0  (₹0)  │◄──────────────────────┐
      │            └──┬──────┬──┘                       │
      │         10/0  │      │  01/0                    │
      │               ▼      └───────────────┐          │
      │        ┌────────────┐   10/0   ┌──────▼─────┐   │
      │ 01/1   │  S5  (₹5)  ├─────────►│ S10 (₹10)  ├───┘
      └────────┤            │          │            │  10/1 , 01/1
               └────────────┘          └────────────┘
                 ▲      │                ▲      │
                 └──────┘ 00/0           └──────┘ 00/0
   Arc label = F T / Open      (F = ₹5 coin, T = ₹10 coin)
```

- Transition list (clearer than any drawing):

| From | FT = 00 (no coin) | FT = 10 (₹5) | FT = 01 (₹10) |
|---|---|---|---|
| S0 (₹0) | S0 / 0 | S5 / 0 | S10 / 0 |
| S5 (₹5) | S5 / 0 | S10 / 0 | **S0 / 1** (₹15) |
| S10 (₹10) | S10 / 0 | **S0 / 1** (₹15) | **S0 / 1** (₹20, change forgiven) |

- Check the examples: 5+5+5 → S5 → S10 → open on the 3rd coin ✓; 10+10 → S10 → open on the 2nd ✓; 5+10 → S5 → open on the 2nd ✓.
- **Moore alternative:** add a 4th state S_OPEN (Open = 1 inside it, next state always S0). Still 2 FFs (no unused code), but the gate opens one cycle **after** the coin arrives.

### (b) State assignment, JK excitation table, equations
- Codes: **S0 = 00, S5 = 01, S10 = 10**, unused **11**. Variables Q1 Q0 F T.

| Q1 Q0 | F T | Q1+ Q0+ | J1 K1 | J0 K0 | Open |
|---|---|---|---|---|---|
| 00 | 00 | 00 | 0 X | 0 X | 0 |
| 00 | 01 | 10 | 1 X | 0 X | 0 |
| 00 | 10 | 01 | 0 X | 1 X | 0 |
| 01 | 00 | 01 | 0 X | X 0 | 0 |
| 01 | 01 | 00 | 0 X | X 1 | 1 |
| 01 | 10 | 10 | 1 X | X 1 | 0 |
| 10 | 00 | 10 | X 0 | 0 X | 0 |
| 10 | 01 | 00 | X 1 | 0 X | 1 |
| 10 | 10 | 00 | X 1 | 0 X | 1 |
| 11 | xx | XX | X X | X X | X |
| any | 11 | XX | X X | X X | X |

- K-maps (rows Q1Q0, columns FT in the order 00, 01, 11, 10; column 11 and row 11 are all X):

| J1 | 00 | 01 | 11 | 10 | | K1 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| 00 | 0 | **1** | X | 0 | | 00 | X | X | X | X |
| 01 | 0 | 0 | X | **1** | | 01 | X | X | X | X |
| 11 | X | X | X | X | | 11 | X | X | X | X |
| 10 | X | X | X | X | | 10 | 0 | **1** | X | **1** |

- **`J1 = Q0'T + Q0F`**, **`K1 = F + T`**.

| J0 | 00 | 01 | 11 | 10 | | K0 | 00 | 01 | 11 | 10 |
|---|---|---|---|---|---|---|---|---|---|---|
| 00 | 0 | 0 | X | **1** | | 00 | X | X | X | X |
| 01 | X | X | X | X | | 01 | 0 | **1** | X | **1** |
| 11 | X | X | X | X | | 11 | X | X | X | X |
| 10 | 0 | 0 | X | 0 | | 10 | X | X | X | X |

- **`J0 = Q1'F`**, **`K0 = F + T`**.

| Open | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | 0 | 0 | X | 0 |
| 01 | 0 | **1** | X | 0 |
| 11 | X | X | X | X |
| 10 | 0 | **1** | X | **1** |

- PIs (all essential): `Q0T`, `Q1T`, `Q1F` → **`Open = Q0T + Q1T + Q1F = T(Q1 + Q0) + Q1F`**.

```
 F ──●──────────┬───────────────────────┐
 T ──┼──●───────┼──────┐                │
     │  │       │      │                │
     │  │   OR(F,T) ───┴──► K1 = K0 = F + T
     │  │
     │  └── AND(Q0',T) ─┐
     └───── AND(Q0, F) ─OR──► J1
     └───── AND(Q1',F) ─────► J0
            AND(Q0,T) ─┐
            AND(Q1,T) ─┼─OR──► Open
            AND(Q1,F) ─┘
        ┌──────┐            ┌──────┐
  J1 ──►│J  Q1 ├──► Q1  J0─►│J  Q0 ├──► Q0      (shared CLK; Q, Q' fed back)
  K1 ──►│K     │      K0 ──►│K     │
  CLK──►│>     │      CLK──►│>     │
        └──────┘            └──────┘
```

### (c) The unused code 11
- **Choice made:** treated as don't-care (next state and Open) to minimise logic. It is never entered in normal operation, and F = T = 1 never happens.
- **Check what the logic actually does from 11** (Q1 = Q0 = 1):
  - No coin: J1 = 0, K1 = 0 → Q1 holds; J0 = 0, K0 = 0 → Q0 holds → **stays in 11**, Open = 0.
  - F: J1 = Q0F = 1, K1 = 1 → Q1 toggles to 0; J0 = 0, K0 = 1 → Q0 = 0 → **00**, Open = Q1F = 1.
  - T: J1 = 0, K1 = 1 → 0; J0 = 0, K0 = 1 → 0 → **00**, Open = 1.
- So 11 behaves **exactly like S10** (₹10 credit): it waits, then opens on any coin and returns to S0. It cannot lock up.
- **Risk:** a power-on glitch into 11 gives a free ₹10 credit. In practice add a reset to force 00 at power-up, or redesign 11 → 00 explicitly (more gates).

**Key Logic & Takeaway**
- Count the distinct "memories" the machine needs → states → ⌈log₂ n⌉ flip-flops.
- Mealy = fewer states, output reacts in the same cycle; Moore = an extra state, output a cycle later but glitch-free.
- Always trace unused codes through the final equations.

---

# BONUS — Past-Paper Sequential Problems (not register-based)

> Two midsem FSM/counter questions in exactly the Tut-7 style, solved with the same method. Worth one read before the exam.

## Midsem 2024-25 Q1 — Counter 4 → 2 → 1 → 0 → 5 → 3 → (4)
- Components allowed: **one T FF, two JK FFs, one 4:1 mux**. Unused states (6, 7) are don't-cares.

| QA QB QC | Next | JA KA | JB KB | TC |
|---|---|---|---|---|
| 100 (4) | 010 | X 1 | 1 X | 0 |
| 010 (2) | 001 | 0 X | X 1 | 1 |
| 001 (1) | 000 | 0 X | 0 X | 1 |
| 000 (0) | 101 | 1 X | 0 X | 1 |
| 101 (5) | 011 | X 1 | 1 X | 0 |
| 011 (3) | 100 | 1 X | X 1 | 1 |
| 110, 111 | X | X X | X X | X |

- K-map results:
  - `JA = QB'QC' + QBQC = (QB ⊕ QC)'` (XNOR), `KA = 1`.
  - `JB = QA`, `KB = 1`.
  - `TC = QA'` (TC = 1 in states 2, 1, 0, 3, all of which have QA = 0).
- **Assignment of the parts:** the T FF for C (`TC = QA'`, a wire from QA'); JK for A and B; the **4:1 mux makes the XNOR**: selects QB, QC, data `1, 0, 0, 1`.

```
            ┌────────────┐
  1 ───────►│I0          │
  0 ───────►│I1  4:1 MUX ├────► JA        KA = 1          ┌────┐
  0 ───────►│I2          │                                │JK  │ QA
  1 ───────►│I3          │                         JA ───►│ A  ├──►
            └──┬──────┬──┘                         1  ───►│    │
              QB      QC   (s1 = QB, s0 = QC)              └────┘
  JB = QA,  KB = 1   ──────────────────────────► JK FF B ──► QB
  TC = QA'           ──────────────────────────► T  FF C ──► QC
  (all three FFs on the same CLK)
```

- **Self-start check:** 110 → (JA = 0, KA = 1 → 0; JB = KB = 1 → 0; TC = 0 → 0) → **000** ✓. 111 → (JA = 1, KA = 1 → toggle → 0; B → 0; C stays 1) → **001** ✓. Both enter the main cycle.

## Midsem 2025-26 Q1 Task C — 2-bit Gray code error-correcting FSM
- The state = last correct angle code (Q1 Q0). The received code is x y (x = B1). The true next code is either the same or the next Gray code (00 → 01 → 11 → 10 → 00). A received code that matches neither is corrected to the valid code at Hamming distance 1. Output z = 1 on error.

| Q1Q0 \ xy | 00 | 01 | 11 | 10 |
|---|---|---|---|---|
| 00 | 00/0 | 01/0 | 01/**1** | 00/**1** |
| 01 | 01/**1** | 01/0 | 11/0 | 11/**1** |
| 11 | 10/**1** | 11/**1** | 11/0 | 10/0 |
| 10 | 00/0 | 00/**1** | 10/**1** | 10/0 |

```
       00/0 , 10/1                       01/0 , 00/1
        ┌──────┐                          ┌──────┐
        │      ▼                          │      ▼
      ┌──────────┐   01/0 , 11/1      ┌──────────┐
      │    00    ├───────────────────►│    01    │
      └──────────┘                    └─────┬────┘
           ▲                                │ 11/0 , 10/1
           │ 00/0 , 01/1                    ▼
      ┌──────────┐   10/0 , 00/1      ┌──────────┐
      │    10    │◄───────────────────┤    11    │
      └──────────┘                    └──────────┘
        │      ▲                          │      ▲
        └──────┘                          └──────┘
       10/0 , 11/1                       11/0 , 01/1
   Arc label = received xy / z   (z = 1 means an error was corrected)
```

- **C2: which flip-flop?** Compute both:
  - D: `D1 = Q0x + Q1x + Q1Q0`, `D0 = Q0y + Q1'y + Q1'Q0` → 12 literals.
  - JK: **`J1 = Q0x`, `K1 = Q0'x'`, `J0 = Q1'y`, `K0 = Q1y'`** → **8 literals** ✓.
  - **Choose JK.** Its don't-cares halve each map.
- Output: `z = Q0'xy + Q0x'y' + Q1'xy' + Q1x'y` (or the equivalent `Q1'Q0'x + Q1'Q0y' + Q1Q0'y + Q1Q0x'`). It is cyclic, with two equally minimal forms.

---

## FINAL CHECKLIST FOR SEQUENTIAL QUESTIONS
- Use the **exact column order and names** the question gives (`D0/J0/K0`, `Q1 Q0`). Marking is often all-or-none per table.
- Excitation table first, then **one K-map per FF input**. Read J from the Q = 0 half and K from the Q = 1 half.
- Mealy vs Moore: output on arcs vs in states. State which one you built.
- Unused states: either make them don't-cares and **trace them**, or force them to a known state.
- Ripple counter direction = (edge type) × (Q or Q').
- Async set/reset acts immediately and overrides the clock; releasing it does nothing until the next edge.

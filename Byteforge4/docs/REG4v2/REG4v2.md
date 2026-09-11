# REG4 — 4-Bit Loadable Register
## AI has been used to enhance deatils and structure 
## What this is

This is the basic storage block I use everywhere in ByteForge-4 — program counter, instruction register, accumulator, general-purpose registers, all of it. Once I had this working, most of the rest of the register-based stuff was just wiring copies of it together.

## Schematic

![REG4 schematic](images/reg4-schematic.png)

## Block Diagram

![REG4 block diagram](images/reg4-block.png)

## Pins

| Pin | Direction | Width | What it does |
|-----|-----------|-------|---------------|
| `D` | Input | 4 | Data going in |
| `LOAD` | Input | 1 | Write enable — tells the register whether to actually store D |
| `CLK` | Input | 1 | Clock, rising-edge triggered |
| `Q` | Output | 4 | Whatever's currently stored |

## How it works

### The storage part: 4 D flip-flops

Nothing fancy — one flip-flop per bit, all four sharing the same clock so they all update at the same instant:

```
D3 D2 D1 D0
│  │  │  │
▼  ▼  ▼  ▼
FF3 FF2 FF1 FF0
│  │  │  │
▼  ▼  ▼  ▼
Q3 Q2 Q1 Q0
```

### The tricky part: making it *loadable*

A bare D flip-flop grabs whatever's on `D` on every single clock edge, whether you want it to or not. That's a problem — most of the time you want a register to just sit there holding its value, not overwrite itself every tick.

The fix is a small trick: feed the flip-flop's own output back into a 2:1 mux, and let `LOAD` pick between "new data" and "my own current value":

```
        ┌─────────┐
D ─────►│ 1       │
        │   MUX   ├──► FF.D ──► FF.Q
Q ─────►│ 0       │
        └────┬────┘
             │
           LOAD
```

- `LOAD = 1` → mux passes `D` → flip-flop captures the new value on the clock edge
- `LOAD = 0` → mux passes `Q` back into itself → flip-flop just re-writes what it already had, i.e. it holds

This is the standard loadable-register pattern — every CPU with registers does some version of this.

## Truth table

| CLK | LOAD | D | Q (next) |
|-----|------|---|----------|
| ↑ | 0 | X | Q (unchanged) |
| ↑ | 1 | 1010 | 1010 |
| ↑ | 1 | 0011 | 0011 |
| ↓ | X | X | Q (unchanged) |
| steady | X | X | Q (unchanged) |

## Building it in Logisim

1. New circuit, name it `REG4`.
2. Drop in input pins: `D` (4-bit), `LOAD` (1-bit), `CLK` (1-bit).
3. One output pin: `Q` (4-bit).
4. Place 4 D flip-flops, all wired to the same `CLK`.
5. Place 4 2:1 muxes — set Data Bits = 1, Select Bits = 1 on each.
6. Wire each mux:
   - top input ← `D[i]` (split off the D bus)
   - bottom input ← `Q[i]` (feedback)
   - select ← `LOAD`
   - output → `FF[i].D`
7. Combine the 4 flip-flop outputs back into `Q` with a 1→4 splitter used in reverse.

### Splitters

- **Input splitter** (for `D`): Fan Out = 4, Bit Width In = 4
- **Output splitter** (for `Q`): Fan Out = 4, Bit Width In = 4, facing West

## Things that'll bite you

- **Mux inputs backwards.** If `D` ends up on the `LOAD = 0` side, the whole register behaves inverted — it holds when you tell it to load and loads when you tell it to hold. Easiest way to catch this: poke `LOAD` by hand and watch which input the mux is actually passing through.
- **Mux isn't 1-bit.** Data Bits has to be 1. Leave it at 4 and you'll get a bus width error.
- **Clock not actually shared.** All four flip-flops need the exact same `CLK` net, or the bits will update at slightly different times and you'll get garbage on fast transitions.
- **`D` floating.** An unconnected `D` pin means the register loads noise. If this happens, check that the wire from the parent circuit is actually reaching REG4's `D` port and isn't dangling somewhere upstream.

## Testing

Test cases and results are in [../tests/REG4_test_notes.md](../tests/REG4_test_notes.md).

## Where this gets reused

- `REG8` — same idea, just twice as wide
- The register file — a bunch of these with address decoding on top
- Program counter — this plus an incrementer
- Accumulator — this fed by the ALU output

## References

- Harris & Harris, *Digital Design and Computer Architecture*, Ch. 5
- Logisim docs: http://www.cburch.com/logisim/docs.html

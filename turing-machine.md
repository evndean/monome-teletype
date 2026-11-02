# Turing Machine

A monome teletype script inspired by the Turing Machine eurorack module by Music Thing Modular.

## Outputs

CV:
1. Primary note. Mirrors the eurorack module by using the first 8 bits.
2. Odd bits. Similar to 1, but only reading the odd bits. May need to include some scaling...
3. Even bits. Similar to 1, but only reading the even bits. May need to include some scaling...
4. (TBD).

Trigger:
1. Primary pulse. Mirrors the eurorack module by using the first bit.
2. (TBD). Could read from a later individual bit, but might be more interesting to read from 2 bits.
3. (TBD). Could read from a later individual bit, but might be more interesting to read from 2 bits.
4. (TBD). Could read from a later individual bit, but might be more interesting to read from 2 bits.

## Script

WIP - sketching out ideas for how this could work.

### Bitwise operators

Some bitwise operators that seem useful:
- `LROT x y` (`<<< x y`) - circular left shift `x` by `y` bits, wrapping around when bits fall off the end.
- `RROT x y` (`>>> x y`) - circular right shift `x` by `y` bits, wrapping around when bits fall off the end.
- `BSET x y` - set bit `y` in value `x`.
- `BGET x y` - get bit `y` in value `x`.
- `BCLR x y` - clear bit `y` in value `x`.
- `BTOG x y` - toggle bit `y` in value `x`.

### Advancing the sequence

In the teletype, each variable normally contains 16 bits; this is the same number of steps in the turing machine.
So, we can use a single variable to store all 16 steps.

If we just want to advance the sequence without changing any values, we can juse use
```
RROT X 1
```
to advance the sequence by 1, assuming we are storing the sequence in the variable `X`.

### Visualizing the sequence

I think it would be nice to be able to see all 16 steps on a grid (or in grid mode on the display).
Some relevant commands:
- `G.CLR` - Clear all LEDs.
- `G.LED x y level` - Set the LED level (0-15).
- `Alt-G` in Live mode will turn on the Grid Visualizer.

We probably don't even need `G.CLR`, because we'll just loop over each LED every time.

So, we can:
1. Loop over all 16 steps.
2. For each step, set the LED as 'on' or 'off'.
    Since the value of each bit will be `1` or `0`, we can use `* (level) (bit value)` to set the level.

```
L 0 15: G.LED I 0 * 10 BGET I X
```

### Introducing randomness

`PARAM` (`PRM`) returns a value in the range 0-16383.
Let's skip the "center is fully random" behavior of the turing machine and instead start with something like:
- knob fully counter-clockwise --> fully random.
- knob fully clockwise --> keep existing note.

`PROB X:` executes a command with probability `x` (0-100).

So, if we scale the param knob from 0 to 100, we can use that, e.g.
```
# do this in the INIT script.
PARAM.SCALE 100 0

# do this in script 1
PROB PARAM: BSET X 0 TOSS
```

### The script

A really basic script could look something like:
1. Circular right shift by 1 bit.
2. Read from the knob to determine probability of setting a new random value.
3. Read and scale the value to determine the CV value.
4. Read the first bit to determine whether to output a trigger.

INIT:
```
PARAM.SCALE 100 0
```

Script 1:
```
RROT X 1
PROB PARAM: BSET X 0 TOSS
CV 1 X
IF BGET X 0: TR.P 1
```

## How a Turing Machine works

It uses a 16-step shift register, which stores the state of 16 bits (individual 0 or 1 values).
The "pulse" output uses the state of the first bit.
The "CV" output uses the state of the first 8 bits, in increasing powers of 2, e.g.

```
01101101

0        --> 0 * 2^0 --> 0 * 1
 1       --> 1 * 2^1 --> 1 * 2
  1      --> 1 * 2^2 --> 1 * 4
   0     --> 0 * 2^3 --> 0 * 8
    1    --> 1 * 2^4 --> 1 * 16
     1   --> 1 * 2^5 --> 1 * 32
      0  --> 0 * 2^6 --> 0 * 64
       1 --> 1 * 2^7 --> 1 * 128

                     --> 182 (out of a maximum value of 255)
```

In the eurorack module, the CV output has a maximum value of ~8.4V.
It also includes a scale knob to restrict this output to a smaller range.

## Resources

- [Turing 201: Turing Machine Explained!](https://youtu.be/va2XAdFtmeU?si=O5ixpayz2ICJwmFz)


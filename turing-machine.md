# Turing Machine

A monome teletype scene inspired by the Turing Machine eurorack module by Music Thing Modular.

If you just want to see the scene itself, scroll to the bottom.

## How the Turing Machine module works

The module uses a 16-step shift register, which stores the sequence in a set of 16 bits (individual 0 or 1 values).
The pulse output is determined the state of the first bit: if the value is high, then a pulse is generated.
The CV output is calculated using the first 8 bits, in increasing powers of 2, e.g.

```
|0|1|1|0|1|1|0|1|
-----------------
|0| | | | | | | | --> 0 * 2^0 --> 0 * 1
| |1| | | | | | | --> 1 * 2^1 --> 1 * 2
| | |1| | | | | | --> 1 * 2^2 --> 1 * 4
| | | |0| | | | | --> 0 * 2^3 --> 0 * 8
| | | | |1| | | | --> 1 * 2^4 --> 1 * 16
| | | | | |1| | | --> 1 * 2^5 --> 1 * 32
| | | | | | |0| | --> 0 * 2^6 --> 0 * 64
| | | | | | | |1| --> 1 * 2^7 --> 1 * 128
-----------------
                              --> 182 (out of a maximum value of 255)
```

In the eurorack module, the CV output has a maximum value of ~8.4V.
It also includes a scale knob to restrict this output to a smaller range.

Don't let the above diagram confuse you about left-shift vs. right-shift bitwise operations.
The variable values in Teletype are 16-bit big-endian values, meaning the largest (most significant) value is on the left, e.g.

```
00000000 00000001 = 1
00000000 00000010 = 2
00000000 00000100 = 4
00000000 00001000 = 8
```

## Working on the teletype scene

### Storing the sequence

In teletype, each variable contains 16 bits (-32768 to 32767).
That is the same number of steps in the Turing Machine module.
This means that we can use a single variable to store all 16 steps in a sequence.
We can then use bitwise operators modify that sequence.

Some teletype bitwise operators that seem useful:
- `LROT x y` (`<<< x y`) - circular left shift `x` by `y` bits, wrapping around when bits fall off the end.
- `RROT x y` (`>>> x y`) - circular right shift `x` by `y` bits, wrapping around when bits fall off the end.
- `BSET x y` - set bit `y` in value `x`.
- `BGET x y` - get bit `y` in value `x`.
- `BCLR x y` - clear bit `y` in value `x`.
- `BTOG x y` - toggle bit `y` in value `x`.

### Advancing the sequence

If we want to advance the sequence and have it loop, we can use circular left shift to advance the sequence by 1:

```
# (assuming we are storing the sequence value in the X variable)
X <<< X 1
```

### Visualizing the sequence

I think it would be nice to be able to see all 16 steps on a grid (or in grid mode on the display).
Some relevant commands:
- `G.CLR` - Clear all LEDs.
- `G.LED x y level` - Set the LED level (0-15).
- `Alt-G` in Live mode will turn on the Grid Visualizer.

Actually, we don't even need `G.CLR`, because we'll just loop over all LEDs in the row every time.
We can do the following:
1. Loop over all 16 steps.
2. For each step, set the LED as 'on' or 'off'.
    Since the value of each bit will be `1` or `0`, we can use `* (level) (bit value)` to set the level.
    (Because of line length limitations, we can only fit a single digit for the level value (i.e. 0 - 9).)

```
# (assuming we are storing the sequence value in the X variable)
L 0 15: G.LED I 0 * 7 BGET I X
```

### Generating an output CV

A few things worth noting:
- Variable values use the Int16 type, with value limits of -32768 / 32767.
- The CV outputs support a range of 0 to 16383 (V 10 - 1).

If we just want the values for the first 8 bits, we can use the bitwise `and` operator with a value that has the first 8 bits set, i.e.

```
# (assuming X holds our sequence value)
& X 255
```

If we then want to scale that to the full range of the CV output, we could do something like

```
# (assuming Y holds the first-8-bits value from above)
SCL 0 255 0 V 10 Y
```

### Example scene: a simple looping CV sequence

This scene will:
- Seed a random initial value.
- Use `X` to store the sequence.
- Use `Y` to store the first 8 bits of the sequence.
- Use `Z` to store the scaled output value of `Y`.
- Draw both `X` and `Y` on the first two rows of the grid

#### M

```
$ 1
```

#### I

```
L 0 16: X <<< X 1; X + X TOSS
```

#### 1

```
X <<< X 1
Y & X 255
Z SCL 0 255 0 V 10 Y
CV 1 Z
L 0 15: G.LED I 0 * 7 BGET X I
L 0 15: G.LED I 1 * 7 BGET Y 1
```

### Adding a trigger output

The Turing Machine module generates an output if the first bit is high.
We can read that value using the `BGET` command:

```
# (assuming X holds our sequence value)
IF BGET X 0: TR.P 1
```

### Introducing randomness

The Turing Machine module has the following looping behavior:
- At noon, the sequences are random.
- At 5 o'clock, it locks into a repeating sequence.
- At 7 o'clock, it double locks into a repeating sequence twice as long as the 'length' setting.
- At 3 o'clock or 9 o'clock, it slips; looping but occasionally changing notes.

That might be a bit complicated to program in a simple, 6 line Teletype script.
Let's start with something simpler: have the param knob (read with `PARAM`, or `PRM`) set the probability that the new value is random, i.e.
- At minimum value (0; fully clockwise; 7 o'clock), loop the sequence (no randomness).
- At maximum value (16383; fully counter-clockwise; 5 o'clock), make the sequence fully random.

`PARAM` (`PRM`) returns a value in the range 0-16383.
`PROB X:` executes a command with probability `x` (0-100).
So, if we scale the param knob from 0 to 100, we can use that, e.g.

```
# (in the INIT script)
PARAM.SCALE 0 100

# (in our sequence generation script, assuming X holds our sequence value)
PROB PARAM: (TODO logic to set the random value)
```

Now we need the logic to set the random value.
`TOSS` randomly returns a `0` or `1` value, which are equivalent to clearing or setting the first bit in a number.
So, we can use `BCLR` to zero out the first bit, then add the TOSS value.

```
# (assuming X holds our sequence value)
PROB PARAM: X BCLR X 0; + X TOSS
```

Unfortunately, that's too long to write on a single line.
As an alternative, we can do something like this:

```
J TOSS
PROB PRM: X BCLR X 0; X + X J
```

### Example scene: introducing randomness

This scene will:
- Use `X` to store the sequence.
- Use `Y` to store a copy of the initial sequence (so we can confirm that it changes over time).
- Draw both values on the first two rows of the grid

#### M

```
$ 1
```

#### I

```
PARAM.SCALE 0 100
L 0 16: X <<< X 1; X + X TOSS
Y X
G.CLR
L 0 15: G.LED I 0 * 7 BGET X I
L 0 15: G.LED I 1 * 7 BGET Y I
```

#### 1

```
X <<< X 1; Y <<< Y 1; J TOSS
PROB PRM: X BCLR X 0; X + X J
L 0 15: G.LED I 0 * 7 BGET X I
L 0 15: G.LED I 1 * 7 BGET Y 1
```

## Final Turing Machine scene

### Simple, single channel sequence

This scene will generate a single CV and single gate output, closely mirroring the behavior of the Turing Machine module.
Additional notes on how the module works can be found in a later section.

Notes:
- The param knob will determine whether the sequence is looping.
    The value of the knob determines the probability that a random value is used.
    - Minimum (fully counter-clockwise; 7 o'clock) means the sequence is looping.
    - Midway (12 0'clock) means there is a 50% chance the sequence will be looped.
    - Maximum (fully clockwise; 5 o'clock) means the sequence will use a random number.
- `X` will store the number representing the sequence.
- `Y` will store the number representing the first 8 bits of the sequence.
- `Z` will store the scaled output value of `Y`.
- The first row of a connected grid (or the grid mode on the screen) will show the sequence.

#### M

```
$ 1
$ 2
```

#### I

```
PARAM.SCALE 0 100
X 0
G.CLR
```

#### 1

```
X <<< X 1; J TOSS
PROB PRM: X BCLR X 0; X + X J
Y & X 255
Z SCL 0 255 0 V 10 Y
CV 1 Z
IF BGET X 0: TR.PULSE 1
```

#### 2

```
L 0 15: G.LED I 0 * 7 BGET X I
L 0 15: G.LED I 1 * 7 BGET Y I
```

## Ideas for extensions

- Update param behavior to more closely mirror the Turing machine.
- Add ability to change loop length (would need to decide what param controls).
- Have 4 independent CV/gate sequences running on CV/gate outputs 1-4.
- Have a single sequence, but read from it in different ways to derive different outputs.
    - CV outputs could use different combinations of bits, e.g. odds, evens, fibonacci, etc.
    - Gate outputs could use a combination of bits, similar to the Pulses expander.
- Scale the CV output(s) to a smaller, more musical range.
    Maybe use the grid to set the range (i.e. one slider for minimum value, one for maximum value).
- Quantize the CV outputs.

## Resources

- [Teletype manual](https://monome.org/docs/teletype/manual/)
- [Turing 201: Turing Machine Explained!](https://youtu.be/va2XAdFtmeU?si=O5ixpayz2ICJwmFz)


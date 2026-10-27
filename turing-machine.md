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


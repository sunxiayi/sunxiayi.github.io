---
title: Two’s Complement of a Signed Binary Number
date: 2021-03-30
categories: CS
tags:
- Binary
- Bits
---

## What

Used for representing signed binary numbers. Invert bits then add 1. For example, 12 in binary representation:

```
0000 0000 0000 0000 0000 0000 0000 1100
```

Invert the digits.

```
1111 1111 1111 1111 1111 1111 1111 0011
```

And add one.

```
1111 1111 1111 1111 1111 1111 1111 0100
```



## Compared with One's Complement

For One's Complement, to get the negative counterpart, we only need to invert the digits. Compared to this method, two's complement has the following pros:

- In One's Complement 0 has two representations(+0 and -0). For example in two bits, `00`->+0, `01`->+1, `10`->-1, `11`->-0. With n bits it can represent 2^n-1 numbers, with 0 being representated twice. In Two's Complement, `00`->+0, `01`->+1, `10`->-2,  `11`->-1. 0 is only representated once, and n bits can represent 2^n numbers. This is because for 0(taking 2 bits as an example): `00` invert digits-> `11` add one-> `100` ignore the overflow and just take 2 bits -> `00`.
- When doing arithmetic operations, One's Complement requires an "add 1" at the end when dealing with negative numbers. For example, 2-1=1 in One's Complement: `10+10=100` taking 2 bits->`00`,  and we need to "add 1" so that `00+01=01`. But for Two's Complement, since there is only one representation of 0, we can skip the "add 1" procedure. In this example, `10+11=101` taking 2 bits->`01` is exactly the answer we want.



## Why Inversion and Adding One Works

Take number "75"(`1001011`) as an example. 

To get the negative counterpart, we subtract 75 from 0 and get:

```
  00000000
- 01001011
-----------
  10110101
```

Which is the same as taking the last 8 bits in:

```
  100000000
- 001001011
------------
  010110101
```

Since `100000000 = 11111111 + 1`:

```
         1
+ 11111111
- 01001011
-----------
  10110101
```

Note that **in binary, when we subtract a number A from a number of all 1 bits, what we're doing is inverting(~) the bits of A.** Thus this operation is equivalent to inverting the number and then add 1.



## References

- [signed binary numbers](https://www.electronics-tutorials.ws/binary/signed-binary-numbers.html)
- [two's complement](https://www.cs.cornell.edu/~tomf/notes/cps104/twoscomp.html)
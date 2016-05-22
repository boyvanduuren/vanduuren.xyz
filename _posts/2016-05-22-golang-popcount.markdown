---
title:  "Population count exercise in GOPL"
date:   2016-05-22 10:32:13+02:00
description: Counting 1's in Go
---

<script src='https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML'></script>

I've been reading [The Go Programming Language][1] by Alan Donovan and Brian Kernighan. Having never read (I know, shame) [The C Programming Language][2] by Kernighan and the late Dennis Ritchie, this seemed like a nice way to get to know Kernighan's (and of course, Donovan's) writing. The book has not been letting me down!

Anyway, some way into the book, section 2.6.2 to be precise, an example is given of a program that counts the population of an integer, meaning it returns the amount of \\(1\\)'s in the binary representation of that integer. <br>
Not intuitively getting how this program worked, it needed some extra attention from my side. Posting this because maybe, somewhere, sometime, someone has the same problem and needs some help.

```go
// gopl.io/ch2/popcount
package popcount

// pc[i] is the population count of i.
var pc [256]byte

func init() {
	for i := range pc {
		pc[i] = pc[i/2] + byte(i&1)
	}
}

// PopCount returns the population count (number of set bits) of x.
func PopCount(x uint64) int {
	return int(pc[byte(x>>(0*8))] +
		pc[byte(x>>(1*8))] +
		pc[byte(x>>(2*8))] +
		pc[byte(x>>(3*8))] +
		pc[byte(x>>(4*8))] +
		pc[byte(x>>(5*8))] +
		pc[byte(x>>(6*8))] +
		pc[byte(x>>(7*8))])
}
```
This program can be split into two parts, the `init()`, and <br> `PopCount(x uint64)`.

So what happens just before, and then in the `init()`? <br>
Well, first of all, we declare an array of 256 bytes, all zero-valued.
Then we iterate through every index and set the value of that element to the population count of its index, meaning that for example `pc[5]` should have the value \\(2\\), because \\(5=101b\\), etc. <br>
How does `init()` do that?

The initialization works around the principle that we can create all the binary numbers using the following rules:

* \\(\\{0, 1\\} \subset B\\)
* \\(x \in B \land x \not= 0 \rightarrow x0 \in B, x1 \in B\\)

So three steps of the second rule will result in:

```html
0,    1
      10,                     11
      100,        101,        110,        111
      1000, 1001, 1010, 1011, 1100, 1101, 1110, 1111
```

We see that every number can be defined by appending either a \\(0\\) or a \\(1\\) to
every other binary number. What happens to a number when we do that?

Appending a \\(0\\) doubles
a binary number, and then we either add \\(0\\) or \\(1\\).
From this follows that when we know how many \\(1\\)'s a binary number \\(n\\) contains, we also know \\(2n\\) contains the same amount of \\(1\\)'s and \\(2n+1\\) has one more \\(1\\).

`init()` actually does this exact same thing. It looks for the number of \\(1\\)'s in `i/2`. Because `i` is an integer halving it always results in a (rounded down) whole number.
Now every 2 iterations (because e.g. `2/2 == 3/2` for integers) we'll lookup the same value, add 0 or 1, and assign that to `pc[i]`.
This can be rewritten mathematically:

* \\(f(0) = 0\\)
* \\(f(n) = f(\lfloor n/2 \rfloor) + n \mod 2\\)


<br>
Having created the lookup table we can now look at the second part, `PopCount(x uint64)`. <br>
For every number that can be represented in a byte we know how many \\(1\\)'s it contains. So we just lookup a number byte for byte and add the results of those lookups. Say we want to check the population count of the number \\(51318=1100 1000 0111 0110b\\), we call `PopCount(51318)`:

```go
// called with x = 51318
func PopCount(x uint64) int {
// x >> 0*8 shifts 0 places to the right, so we still have 0000 ... 1100 1000 0111 0110
// byte(x>>(0*8)) takes the least significant byte of that number, resulting in 0111 0110b = 118
// pc[118] = 5
	return int(pc[byte(x>>(0*8))] + // = 5

// x >> 1*8 shifts 8 places to the right, so we have 0000 ... 0000 0000 1100 1000
// byte(x>>(1*8)) again takes the LSB, resulting in 1100 1000b = 200
// pc[200] = 3
		pc[byte(x>>(1*8))] +        // = 3
        
// x >> 2*8 = 0000 ... 0000
// byte(x>>(2*8)) = 0000b = 0
		pc[byte(x>>(2*8))] +        // = 0

/// same as x>>2*8 for the next few steps
		pc[byte(x>>(3*8))] +        // = 0
		pc[byte(x>>(4*8))] +        // = 0
		pc[byte(x>>(5*8))] +        // = 0
		pc[byte(x>>(6*8))] +        // = 0
		pc[byte(x>>(7*8))])         // = 0

// returns 5+3+0+0+0+0+0+0 = 8
}
```

This should explain the workings of this algorithm.


As a small extra: <br>
When looking at this program we found that we can represent the initialization as:

* \\(f(0) = 0\\)
* \\(f(n) = f(\lfloor n/2 \rfloor) + n \mod 2\\)

This implies we could also do a population count using this recursive function:

```go
func PopCount(x uint64) uint64 {
	if x == 0 {
		return 0
	}
	return foo(x/2) + x%2
}
```

Which of course isn't as fast as the lookup method if you have to count more than one number, but it proves all of the above.

[1]: http://www.gopl.io/
[2]: http://www.amazon.com/Programming-Language-Brian-W-Kernighan/dp/0131103628

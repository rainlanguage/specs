# Discussion of gas in Rain

Rain an interpreted language onchain so it sounds impossible that gas costs could
be comparable or less than native compiled code.

This document aims to unpack some assumptions, losses and wins incurred and
achieved by the use of an interpreted language.

## Assembly/optimised implementations in the interpreter and calling contracts

Most Solidity contracts out there are Solidity, not the low level Yul language
provided as part of Solidity.

This is for good reason, the difficulty of verifying correctness of Yul is far
higher than Solidity, but use of Solidity often has a negative impact on gas.

This is because Solidity implements its syntax as "one size fits all" algorithms
that always err on the side of safety. Solidity documentation claims that using
Yul rather than Solidity often gives a 10-15% gas saving, but we've seen as high
as 80%+ savings for certain algorithms.

It's highly unlikely that the target demographic of Rainlang
(beginner to intermediate skill level) would ever write highly optimized
Solidity, so the gas costs of the compiled Rainlang should be compared to the gas
cost of a relatively naive/vanilla Solidity implementation of the same logic.

We can list some things that Solidity does, why it does them and why we can often
avoid doing them to save gas in a safe way.

### Compile time vs. Runtime calculations

A lot of what we need to run a smart contract is known at the moment we deploy it
but Solidity might treat it as a runtime concern. For example, we may want a
"dynamic" array rather than a tuple, with known size at compile time, but
Solidity will still generate a runtime loop to build it.

Usually Solidity does a good job of calculating offsets etc. but there can be
times when it generates unnecessary loops or allocations.

If we know something at compile time that Solidity doesn't we MAY be able to cut
gas by "hardcoding" some logic that would otherwise be calculated at runtime.

### Jumps are expensive

Whenever we have to call an internal function, loop, evaluate an `if` statement,
etc. the code jumps from one point of execution to another.

Solidity autogenarates jumps for a lot of logic, which can ramp up gas costs.

Of course, removing ALL jumps would both destroy ALL code maintainability by
making it impossible to use function calls, and balloon the size of the final
codebase as to probably be undeployable anyway.

However, being aware that jumps cost more than not-jumping can save thousands or
even 10s of thousands of gas on hot performance paths.

### Out of bounds checks are expensive

Solidity knows nothing about why you are asking it to index into, or numerically
manipulate something. Therefore it has to assume that an attacker is attempting
to read/write/calculate something out of bounds, underflowing or overflowing the
valid space of values.

Often we know at compile time that something can't escape the bounds. The
`unchecked` keyword allows overflow/underflow checks to be bypassed for numerical
calculations but doesn't bypass checkes for indexing into data structures like
`array[x]`. It will ALWAYS pay the gas to check that `x` is less than the array
length, which can be especially painful in a loop.

### The `new` keyword zeros out memory structures

Whenever Solidity creates some data structure in memory like `new uint256[](N)`
it will allocate a region of memory for the array then _loop over the newly
allocated memory_ to zero it out. This is because Solidity makes no guarantee
that the region of memory after the free memory pointer is zero before the
allocation, so to make that guarantee after the allocation it MUST loop.

If we do not need the newly allocated memory to be zeroed out, e.g. it is very
common to immediately populate some array with values, then this loop is
useless and costs 70+ gas even for small arrays, and worse for larger arrays.

For example, consider

```solidity
uint256[] memory array = new uint256[](2);
array[0] = 1;
array[1] = 2;
```

vs.

```solidity
uint256[] memory array;
assembly ("memory-safe") {
    array := mload(0x40)
    mstore(0x40, add(array, 0x60))
    mstore(array, 2)
    mstore(add(array, 0x20), 1)
    mstore(add(array, 0x40), 2)
}
```

The former does a few unneccessary things that cost ~730 gas (about 3x total cost):

- Zeros the high bits of the number `2` to convert it between types to use it as
  the length of an array
- Loops over the `new` array to zero it out, invoking jumps in the process
- Checks that the indexes `0` and `1` are not out of bounds of `2`

But the latter is clearly unintelligble without comments and/or careful automated
testing, probably including a fuzzer and checks that the manual memory
allocations are correct.
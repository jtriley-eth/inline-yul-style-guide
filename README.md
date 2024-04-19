# Inline Yul Style Guide

This is a guide for optimizooors developing EVM-based smart contracts. We use inline assembly, a dialect of Yul, when optimizing such contracts, so we call it an "Inline Yul Style Guide". This guide also covers development and testing practices. We are sharing this publicly in case it is useful to others.

## Why?

Coinbase has recently released a [Solidity Style Guide](https://github.com/coinabse/solidity-style-guide), which is an excellent style guide for modern Solidity smart contract development, but its [Section 1.C.4](https://github.com/coinbase/solidity-style-guide?tab=readme-ov-file#4-avoid-using-assembly) states "Avoid Using Assembly".

We believe inline assembly is not only beneficial to users who bear the cost of transaction execution against our contracts, it also improves market efficiency for sophisticated actors that are drawn to lower gas fees given comparable external variables. Sufficient testing, aesthetics, and documentation make inline assembly both viable and secure for production use cases.

## Style

We generally follow the [Coinbase Solidity Style Guide](https://github.com/coinabse/solidity-style-guide) who in turn follow the [Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html) unless they specify otherwise.

### Variable Naming

Variables are named in camel case with minimal abbreviations aside from well known, commonly used abbreviations:

1. `fmp` for "free memory pointer"
2. `ptr` for "pointer"
3. `len` for "length"

### Double Whitespace

Each instruction belongs on its own line and that line belongs between two blank lines. Rare exceptions may be made for brevity when operations perform closely associated actions.

```solidity
// -- snip
let ok := 0x01
let i := 0x00

mstore(0x00, selector)

mstore(0x04, argument0)

for { } 0x01 { } {
    if eq(i, 0x10) { break }

    ok := and(ok, call(gas(), target, 0x00, 0x00, 0x2 , 0x00, 0x00))

    i := add(i, 0x01)
}

stop()
```

### Indentation

Indentation should be minimized. The squint test should be applied here to minimize nesting of indentations. Shallow indentation signals brevity, clean abstractions, and simple control flow.

We make exceptions to convention in favor of readability, particularly with `if` statements containing only a single instruction. This exception is not appliable to `for` loops or `switch` statements.

```solidity
let ok := 0x01

// -- snip

if ok { stop() }

revert(0x00, 0x00)
```

### Avoid Excessive Variables

Convention prefers variables over magic values for readability, though this can get in the way of clean inline assembly and add unnecessary lines of code, imports, and indirection.

Common exceptions to the excessive variables rule are:

- zero (`0x00`)
- one (`0x01`)
- word size (`0x20`)
- free memory pointer slot (`0x40`)

However, other cases such as selectors, unknown offsets and pointers, and unusual literals should be assigned to constants when possible.

> Note that Solidity limits constants that can be used in Yul to be literals. Creating a constant out of a keccak hash only works if you specify the literal, not call the keccak function. We hope this changes in the future.

```solidity
uint256 constant balanceOfSelector = 0x70a0823100000000000000000000000000000000000000000000000000000000;

// -- snip

mstore(0x00, balanceOfSelector)

mstore(0x04, caller())

let success := staticcall(gas(), target, 0x00, 0x24, 0x00, 0x20)

success := and(success, eq(returndatasize(), 0x20))

if iszero(success) { revert(0x00, 0x00) }

return(0x00, 0x20)
```

### Instruction Composition

When instructions depend on the output of other instructions it is natural to want to nest them inside one another. This is a problem because when the line length would overflow, we break it into multiple lines, hurting readability and unnecessarily increasing indention. Additionally, instruction arguments are evaluated form right to left, which makes code difficult to reason about when instructions perform side effects.

Consider a snippet from a common `SafeTransferLib` implementation:

```solidity
// -- snip
success := and(
    or(and(eq(mload(0), 1), gt(returndatasize(), 31)), iszero(returndatasize())),
    call(gas(), token, 0, freeMemoryPointer, 68, 0, 32)
)
// -- snip
```

This can be improved significantly by decomposing the instructions into separate lines:

```solidity
let success := call(gas(), token, 0, fmp, 68, 0, 32)

let returnedTrue := and(eq(mload(0), 1), gt(returndatasize(), 31))

let returnedNothing := iszero(returndatasize())

success := and(success, or(returnedTrue, returnedNothing))
```

Known optimizer rules also allow for more aesthetic code. This is especially applicable when using basic arithmetic, the compiler checks for mathematical optimizations at compile time, giving zero code aesthetics.

```solidity
let fmp := mload(0x40)

mstore(add(fmp, 0x00), item0)
mstore(add(fmp, 0x20), item1)
mstore(add(fmp, 0x40), item2)
mstore(add(fmp, 0x60), item3)
```

### Grouping Associated Actions

Associated actions should be grouped together. This is also a product of the squint test, the colors printed by the syntax highlighter should be closely grouped together when possible. This allows for cleaner reasoning about clusters of actions.

```solidity
// -- snip

let ok := 0x01

let i := 0x00

let fmp := mload(0x40)

mstore(add(fmp, 0x00), item0)

mstore(add(fmp, 0x20), item1)

for { } 0x01 { } {
    if eq(i, 0x10) { break }

    ok := and(ok, call(gas(), target, 0x00, 0x00, 0x2 , 0x00, 0x00))

    i := add(i, 0x01)
}
```

### Document The Shit Out of Everything

Documentation is one of the most important components in writing inline assembly. Clear documentation should be associated with every line of assembly. However, the way documentation is written is also important. Interleaving lines of code with lines of documentation interrupts the flow of reading each. But grouping each together makes for a clean one to one mapping without interrupting the flow.

```solidity
// -- snip

// Query ERC20.totalSupply without allocating new memory.
//
// Procedures:
//      01. store totalSupply selector in memory
//      02. staticcall totalSupply; cache as ok
//      03. check that the return value is 32 bytes; compose with ok
//      04. if ok is false, revert
//      05. assign returned value to output
function totalSupply(ERC20 erc20) view returns (uint256 output) {
    assembly ("memory-safe") {
        mstore(0x00, totalSupplySelector)

        let ok := staticcall(gas(), erc20, 0x00, 0x04, 0x00, 0x20)

        ok := and(ok, eq(returndatasize(), 0x20))

        if iszero(ok) { revert(0x00, 0x00) }

        output := mload(0x00)
    }
}
```

## Testing

Inline assembly performs next to no implicit behavior. No checks are performed on the code, so we have to check everything for it.

A unit test should cover each:

- branch of code
- invalid arithmetic
- invalid data location access

A fuzz test should cover every entry point with each unit test's cases taken into account.

```solidity
function testTotalSupply() {}

function testTotalSupplyContractDoesNotExist() {}

function testTotalSupplyThrows() {}

function testTotalSupplyInvalidReturndatasize() {}

function testFuzzTotalSupply(bool exists, bool shouldThrow, bool shouldReturnValidData) {}
```

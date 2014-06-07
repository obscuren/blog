Mutan is a language that is designed for the [Ethereum](http://github.com/ethereum/go-ethereum) platform and enables users to create and develop decentralised contracts which are stored within Ethereum's block chain. Without going in to too much detail about what Ethereum is and does; think of it as a decentralised consensus network with a Virtual Machine. The VM can execute so called contracts (pieces of code) that are stored within the block chain when triggered.

Mutan's basic syntax resembles that of Go but, underneath is much more like the C language. It attempts to abstract some of the lower level aspects of the language but does not do so by lifting control from the programmer.

```go
func twoTimes(var a) {
    return 2*a
}
var x = twoTimes(42)
```

The full language specification can be found [here](https://github.com/ethereum/go-ethereum/wiki/Mutan-0.2)

## Variables

Variables in Mutan are all variadics, meaning that they can be of any type at any time. Mutan knows 4 basic types; numerics, strings, arrays and pointers.

Strings are simple words enclosed in quotes

```go
"hello"
```

Numerics can be either hexadecimal values or decimals

```go
123
0xabc
```

Assigning a value to a variable can be done in two ways; with the explicit `var` keywords followed by the identifier or by using the `:=` initialisation operand.

```go
var x = 0x1
y := 1
```

Arrays are pre-allocated chunks of memory with a size specified before compile-time (i.e., static size arrays)

```go
var[10] a
a[2] = 10
```

Pointers can be created by passing variables around as an address. If you don't know what pointers are think of them like this:

```
When I give you an URL of a website I give you a _pointer_ to the website (value) where you and I can both see the changes if they so happen to be made.

When I give you a printed version on paper of the website I give you the value of the pointer instead where changes made to the website are not reflected on paper.
```

```go
var a = 10
var *b = &a
```

The `&` operand takes the address of variable a and assigns it at b. b Now points at variable a.

Imagine you want to write a function that changes a value by multiplying it by 2. The changes made need to be reflected in the original variable.

```go
func timesTwo(var a) {
    a = a * 2
}
var a = 10
timesTwo(a)
```

In the above function the `var a` declared on line 4 will still be `10` even after `timesTwo` has been called. What happens here is that when timesTwo was called it read out `a` and passed it as a `value` (the printed copy of the website). Changing `a` on line 2 is therefor not reflected in the original variable on line 4.

If we'd need this type of behaviour we should have written it in a different fashion using pointers

```go
func timesTwo(var *a) {
    *a = *a * 2
}
var a = 10
timesTwo(&a)
```

This time `var a` will be `20`. So how does this work? On line 5 we didn't pass the value along but the address of `a` instead allowing `timesTwo` function to read out the address the pointer is pointing at (like the URL) and reading / writing anything it's pointed at. Reading a pointer is done by **dereferencing** the pointer with the `*` operand. `*a` basically says; give me whatever `a` is pointing at, which so happen to be 10 in this case. Therefor **assigning** anything at `*a` means, set _x_ where `a` is pointing at.

## Functions

In the previous topic you've seen we can define new functions using the `func` keyword. Apart from calling functions and passing variables around they can also have return values. Values are being returned using the `return` keyword. If a function returns a value the function signature `"func" identifier "(" [ args ] ")"` needs to have an extra `var` at the end telling the compiler the function returns a value.

```go
func fn() var {
	return 10
}
```

This tells the compiler "function fn returns a value" and allows it do some error checking such as:

* does the function body return a value
* does the function assign to a variable

## Buildin functions

Mutan comes with a couple of build in functions that call some Ethereum specific opcodes.

```
data          Returns the x'th value of the attached data of this call
dataSize      Returns the size of the data attached to this call
origin        Returns the origin address of this execution
caller        Returns the current caller of the closure
gasPrice      Returns the gas price attached to this call
value         Returns the value attached to this call
balance       Returns the value of the current call
diff          Returns the current difficulty
prevHash      Returns the previous block's hash
time          Returns the current block's timestamp
gasPrice      Returns the attached call's gas price
number        Returns the current block's number
coinbase      Returns the current block's coinbase
gas           Returns the current call's attached amount of gas
```

The buildin functions can be called on the `this` object: `this.data[0]`.

## Assembler

Using the `asm` keyword mutan allows you to directly use E-ASM (Ethereum Assembler) giving the programmer the tools to get down to nitty gritty and optimise where it sees fit. 

```go
asm(
	PUSH 0  ; Push stack ptr mem offset
    MLOAD   ; Load into VM stack
)
```

There're also two helper functions which will allow you to `push` and `pop` values from and to the VM stack. These functions are especially handy in combination with the `asm` keyword.

```
m_push(expr)   pushes the _expr_ to the stack.
m_pop()        creates a ignore instruction to use with return which expects a value.
```

Say you want to write a function that uses `asm`. The asm code needs to use one of the functions's variables. You could load the stack ptr, increment it by the offset of the variable in order to load the value. Instead you'd use the `m_push` buildin that does all that for you.

Imagine you want to write a sha3 function. Ethereum has the SHA3 opcode which takes two items of the VM stack; offset in memory and the size of the memory it needs to read. By leveraging asm and the push, pop buildins we can easily write that in mutan itself.

```go
func sha3(var *v, var size) var {
	m_push(size)    // push the size on to the VM stack
    m_push(v)       // push the offset (ptr)
    asm(SHA3)       // perform sha3
    
    return m_pop()  // Return the value of SHA3
}
```

## Memory

Mutan's memory model is much like that of C, such that it uses a stack based approach, but, without limitations. The stack can grow as large as memory allows simply because there's no dynamic memory allocation. Functions such as C's `malloc` aren't present at this time of writing.

Each function call in Mutan creates a new **stack frame** which is the scope of that current executing call. A scope or stack frame can be referenced by looking at the `___stackPtr`. The stack pointer points at the current stack frame allowing variables to be declared on the right stack frame. The stack pointer is a special type of variable that is initialised at the beginning of each execution and is initialised to `nil` by default. Every time a new stack frame is created you take the size of stack and add that on top of the current stack pointer's value (which is the point of the previous stack).

Mutan's memory model can be illustrated as followed

```
000: ___stackPtr
--- Stack frame 1
001: "___retPos"
002: "___stackSize"
003: Stack var "a"
004: Stack var "b"
005: Stack var "c"
```

Each function call also defines two values by default; the `___retPos` and the `___stackSize`. The retPos or return position is being set _before_ executing the function and sets it to the current **program counter** (PC). You can think of the PC as the current point in code from where it's executing. The retPos is used to determine the return position in code when encountering a `return` statement. The stackSize variable or previous stack size variable is used to set -once returning from a function call- the previous stack pointer. Think of it as the `pop`ing the frame of the stack. Once the stack pointer has been set anything **above** it is marked as _free space_.

```
+ indicates in use
- indicates empty / not is use
011 -                              011 -
010 + END   F2                     010 -
009 +                              009 -
008 +                              008 -
007 +                              007 -
006 +                              006 -
005 + BEGIN F2       After pop     005 -
004 + END   F1                     004 + END f1
003 +                              003 +
002 +                              002 +
001 + BEGIN F1                     001 + BEGIN F1
000 + PTR005                       000 + PTR001
```

The stack ptr (`000`) points at memory address `005` marking the current stack frame at `005` up to `010` (i.e. frame 2). By popping stack frame 2 we set the stack ptr to `001` to `004`, marking anything above that as free.

***

* [Comments](http://www.reddit.com/r/Mutan/comments/27jjou/mutan_02/)
* [Mutan 0.2](https://github.com/obscuren/mutan)
* [Ethereum](https://github.com/ethereum/go-ethereum)

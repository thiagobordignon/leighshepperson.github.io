---
layout: post
title:  "Let's Build a Register Based Virtual Machine in Rust: Part 1"
date:   2016-06-13 23:21:31 +0100
categories: jekyll update
---
### Introduction

In this post, we'll create a simple register based virtual machine and in Part 2 we'll use it to compute the greatest common divisor of a pair of small integers. 

Virtual machines are usually written in system level languages, so that's why we're going to build the virtual machine using Rust.

We won't cover either Rust or Register machines in great detail, so if you need more details, have a read of the [Rust book](https://doc.rust-lang.org/book/README.html) and [Compilers: Principles, Techniques, and Tools](https://www.amazon.co.uk/Compilers-Principles-Techniques-V-Aho/dp/1292024348).

### Registers

In a physical processor, a *register* is an area of data storage available to a central processing unit (CPU). They are used to speed up programs so they don't have to access the main memory to do simple operations like add and subtract.

Our virtual machine needs four virtual registers: An *instruction pointer* `ip` to keep track of the next instruction in the program and three arithmetic registers in a *register file* called `registers`. Note, a *register file* is just an array of registers. We'll initialise these inside the `cpu` function:

{% highlight rust %}
type RegisterFile = [u16; 3];

fn cpu() {
    let mut ip = 0;
    let mut registers = RegisterFile::default();
}
{% endhighlight %}

Let's talk briefly about the implementation.

Since `RegisterFile` is a type alias of `[u16; 3]` it has access to all its inherent methods. So the line

{% highlight rust %}
let mut registers = RegisterFile::default();
{% endhighlight %}

just calls the default function on `[u16; 3]` giving

{% highlight rust %}
registers = RegisterFile::default() = [0, 0, 0];
{% endhighlight %}

Additionally, if the registers were declared as global mutable variables outside the `cpu` function, then other functions or methods could use them without passing them in as parameters. However, this is unsafe in multithreaded applications, so the Rust language forces you to use the `unsafe` code block in this situation. For example, if you wanted to print out the value of a global version of the instruction pointer, then you would have to do it like this:

{% highlight rust %}
unsafe {
    println!("{:?}", ip);
}
{% endhighlight %}

### The Fetch, Decode, Execute Cycle

The `cpu` function performs the fetch, decode, execute cycle. First, the instruction pointer **fetches** an instruction from the instruction set. Here, the *instruction set* is an array of instructions encoded as 16 bit words, such as `[0xFF0D, 0xAB0A]`. Then, the instruction is **decoded** and the CPU **executes** it by performing an appropriate action. Finally, the instruction pointer is incremented and the process continues until it halts.

![Fetch Decode Execute](/uml/dc1775cd274cabde5881553e43f4a981.png)

We'll give more details in the following sections:

### Instructions

The virtual machine reads from an array of instructions encoded as 16 bit words. For example:

{% highlight rust %}
let encoded_instructions = [0xFFFD, 0xFF2A, 0x1121];
{% endhighlight %}

An *opcode* is the part of the encoded instruction that specifies the operation that gets performed. In our encoding, the first 4 bits of an encoded instruction represent the opcode; the remaining bits specify the parameters.

The following table defines the opcodes used by our virtual machine:

| Opcode        | Description           |
| ------------- |-------------|
| Halt      | terminates the program and prints out the value of the 0th arithmetic register |
| Load      | loads an 8 bit unsigned integer into the destination register      |
| Swap | swaps the data in the destination register with the data in the source register.  You must specify a temporary register   |
| Mod   | computes the modulus of two register values and puts the result in a new register   |
| BEZ   | moves the instruction pointer back by offset-many places only if the register's value is equal to zero


<br />
The following table defines the instruction encoding:

| Opcode   | 1st hex digit   | 2nd hex digit   | 3rd hex digit   | 4th hex digit   |
| ------------- |-------------| ------------- |-------------| -------------| 
| Halt   | 0   | -   | -   | -   |
| Load   | 1   | destination register   | value   | *   |
| Swap   | 2   | destination register   | source register   | temporary register   |
| Mod   | 3   | dividend  | divisor  | destination   |
| BEZ   | 4   | register   | offset   | *   |

<br />
Here, we use "*" to mean the hex digits to the right are part of the same value. For example, the value in the **Load** instruction is an 8 bit integer, not two 4 bit integers. The "-" means we ignore that hex digit.

In Rust, enum variants can have data associated to them. So we use an enum to represent instructions, where each variant represents the opcode and its data are the parameters it operates on:

```rust
enum Instruction {
    Halt,
    Load { destination: usize, value: u16 },
    Swap { destination: usize, source: usize, temp: usize },
    Mod { dividend: usize, divisor: usize, destination: usize},
    BEZ { register: usize, offset: usize }
}
```


### Fetch

To implement the *fetch* operation, we'll first define the `Program` struct:

{% highlight rust %}
struct Program<'a> {
    instructions: &'a [u16],
}
{% endhighlight %}

The `instructions` field is an array slice that uses an explicit `'a` lifetime. Here, an *array slice* is an array whose size isn't known at compile time and the *explicit lifetime* `'a` means the `Program` instance won't outlive the referenced `instructions` object. 

Now we give an implementation of `Program` that defines a *fetch* method that returns the instruction indexed by the instruction pointer:

{% highlight rust %}
impl<'a> Program<'a> {
    fn fetch(&self, ip: usize) -> u16 {
        self.instructions[ip]
    }
}
{% endhighlight %}

We use the fetch method like so:

{% highlight rust %}
let encoded_instructions = Program { instructions: &[0x1110, 0x2100, 0x3010, 0x0] };

let encode_instruction = program.fetch(0);
{% endhighlight %}

### Decode

Since the instructions are encoded hexidecimal words, we'll need a simple way of decoding them. We can do this by using bit shifts and masks. 

For example, consider the encoded instruction `0x13D2`. Writing this out in binary gives us the following representation:

```rust
0001 0011 1101 0010
```

We can easily find the opcode **Load** by bit shifting it 12 places to the right:

```rust
        0001 0011 1101 0010 = 0x13D2 
>> 12   0000 0000 0000 0001 = 0x1 = 1
```

Since the operation is **Load**, we know from the encoding table that the next 4 bits represent the register and the last 8 bits must be the value. To get the register we need to shift `0x13D2` by 8 bits to the right and then apply the bit mask `0xF` to kill off the unwanted bits:

```rust
        0001 0011 1101 0010 = 0x13D2
>> 8    0000 0000 0001 0011 = 0x13
    
        0000 0000 0001 0011 = 0x13
        0000 0000 0000 1111 = 0xF
AND     0000 0000 0000 0011 = 0x3 = 3
```

Finally, we get the value by performing the bitwise **AND** operation with `0xFF` and converting from hexidecimal to decimal:

```rust
        0001 0011 1101 0010 = 0x13D2 
        0000 0000 1111 1111 = 0xFF
AND     0000 0000 1101 0010 = 0xD2 = 210
```

In our virtual machine, we implement the decode step by giving an implementation of `Instruction` that defines a `decode` method that returns the appropriate `Instruction` variant. If we are not able to decode the instruction, then we'll use the generic `Return` type to return the variant `Err` with an informative message. Otherwise, it just returns `Ok` and its value.

```rust
impl Instruction {
    fn decode(encoded_instruction: u16) -> Result<Self, &'static str> {
        let operator = encoded_instruction >> 12;
        let reg1 = ((encoded_instruction >> 8) & 0xF) as usize;
        let reg2 = ((encoded_instruction >> 4) & 0xF) as usize;
        let reg3 = (encoded_instruction & 0xF) as usize;
        let value = encoded_instruction & 0xFF;

        match operator {
            0 => Ok(Instruction::Halt),
            1 => Ok(Instruction::Load { destination: reg1, value: value }),
            2 => Ok(Instruction::Swap { destination: reg1, source: reg2, temp: reg3 }),
            3 => Ok(Instruction::Mod { dividend: reg1, divisor: reg2, destination: reg3 }),
            4 => Ok(Instruction::BEZ { register: reg1, offset: value }),
            _ => Err("Failed to decode the instruction")
        }
    }
```

### Execute

The execution step involves performing an arithmetic operation on one or more of the registers. For example, let's consider the **Swap** operation:

In our virtual machine, there's a register file containing three arithmetic registers. Now let's say we want to swap the data in register 1 with the data in register 2:

![Fetch Decode Execute](/uml/6b842a1b925005098bac06c3f32addf0.png)

First, we'll copy the data of register 1 into a temporary register (which in our case is register 3):

![Fetch Decode Execute](/uml/417d34727c6d05050c0c1731d572afe3.png)

Then, we'll overwrite the data in register 1 with the data in register 2:

![Fetch Decode Execute](/uml/996d1ac39f77b5c02cdfc35b216ed131.png)

Finaly, we'll copy the data of register 3 into register 2:

![Fetch Decode Execute](/uml/ca85471a4753fa5181b65fdca52a340f.png)

<br />
The **Swap** operation is easily implemented like so:

```rust
fn swap(source: usize, destination: usize, temp: usize, registers: &mut [u16]) {
    registers[temp] = registers[source];
    registers[source] = registers[destination];
    registers[destination] = registers[temp];
}
```

Finally, we can extend the implementation of the `Instruction` enum by defining an `execute` method that pattern matches on the `Instruction` variant to determine what happens when it executes:  

```rust
fn execute(&self, registers: &mut [u16], ip: &mut usize) -> bool {
    match *self {
        Instruction::Load { destination, value } => {
            registers[destination] = value;
        },
        Instruction::Swap { destination, source, temp } => {
            swap(destination, source, temp, registers);
        },
        Instruction::Mod { dividend, divisor, temp } => {
            registers[temp] = registers[dividend] % registers[divisor];
        },
        Instruction::BEZ { register, offset } => {
            if registers[register] == 0 {
                *ip -= offset;
            }
        },
        Instruction::Halt => {
            println!("{:?}", registers[0]);
            return false;
        },
    }

    true
}
```

[x86-regs]: http://www.eecg.toronto.edu/~amza/www.mindsec.com/files/x86regs.html 
---
layout: post
title: "Diary of writing a RISC-V assembler"
date: 2026-08-24 12:00:00
tags:
  - assembly
  - RISC-V
  - ELF
  - C
categories:
---

Hello! Here I'm describing my journey about writing [`ras`](https://github.com/thedevbirb/ras), a RISC-V/ELF 32/64-bit assembler from scratch in C, with no dependencies other than `libc`, sharing my learnings and thoughts. There could be good takes, and probably a lot of bad takes, be wary!
The first part of this post focuses a lot on motivation, _my personal experience_ in writing it, and what I'm getting from it. It assumes some knowledge of assemblers already. A second part gives a tour of how an assembler works in practice, with all its quirks and complexities. Pick what you like!

## Why

Let's start first with the most brutal question for the dismissive reader, who may wonder: *"..but, but GNU `as` / LLVM MC already exist!"*. Yes, I know; it's an _okay argument_ but in practice I don't care _much_. I started it because I was looking to build some software to get experience writing non-trivial C code, and to get exposure to assembly; I was lacking on both, which I think they're paramount to know for every software engineer. I've chosen RISC-V since a lot of new hardware (especially microcontrollers) that supports such ISA are getting built, and I thought an almost fixed-size instruction set (16/32-bit) would be simpler to learn than a variable one, like x86. However, [this turned out to not be as accurate for RISC-V](https://dmitry.gr/?r=06.%20Thoughts&proj=12.%20RV). Lastly, I was excited to contribute to open-source software/hardware by understanding it more, and trying to improve the status quo of assemblers by creating a new codebase to experiment with.

This projects has also been built in a delicate period of my life. I've been questioning myself many times on what it means to me being a software engineer and what is means to write (or generate?) good code. I'm trying to pivot my career from writing "service"-oriented code in startups to getting closer to products which require understanding hardware. Overall, I have profound admiration of people who can write powerful products thanks to good direction and [_mechanical sympathy_](https://mechanical-sympathy.blogspot.com/2011/07/why-mechanical-sympathy.html). I want to follow this path and look for such positions, and I thought this could be a nice experiment towards such direction.

If you want to read a really unstructured diary, with day to day notes including minor existential crises (:D), you can find it [here](https://github.com/thedevbirb/ras/blob/main/diary.md).
## What is the value?

Of course, what I've built isn't really comparable with  production assemblers that have decades of work and testing poured into them, so what's point? What is its value?

### Understanding

Probably the most important part is the fact I'm among the _not many_ humans who can reason about such how such software works. An assembler, like a compiler, is a non-trivial software that requires a decent amount of context about how computers and operative systems work together, and which format do they communicate with.
I have no current plans nor idea on how to extract any economic value from this project, but I think such knowledge can be helpful to some companies, or a prerequisite to create something new. Overall, since assemblers have been around for more than 40 years, it means there is demand for people who can understand it, improve it, and perhaps guide new directions after learning past lessons.

In short, there is value in having context about a certain domain, after you've had some hands-on experience with it.

### Simplicity

There is also some value in its simplicity: while one could say it's less powerful because it only supports one architecture/object file combo, the upside is that you have a much smaller footprint to work with. If you ever read GNU `as` or LLVM MC you can get overwhelmed very quickly by the amount of both generic _and_ target specific code, either via preprocessing logic or class indirections, which makes reasoning on the codebase much harder. I'm more familiar with GNU `as` and there examples include:

- The core logic and components (`expr.c`, `read.c`, `write.c`, `frags.c`) still contain a lot of target/object specific code, yet architecture specific code like `tc-riscv.c` might contain workarounds or suboptimal implementations to adhere with the common interfaces. Here, the slowest example consists of [`GAS_SORT_RELOCS`](https://github.com/gnutools/binutils-gdb/blob/8818c47980038800239da822cfef2aa63e0fac1f/gas/write.c#L1370-L1396), which is enabled for the RISC-V target, and adds a time consuming sorting implementation of the relocations inside the object file. When assembling the amalgamation of the SQLite 3, consisting of ~260k lines of assembly, it takes about ~80ms (M4 Pro) to sort 67k relocations. This step is entirely avoided in `ras` by saving fixup/relocation information in a doubly linked list and preserving order at insertion time with very easily.
- The overhead of `BFD`, the Binary File Descriptor library, which is GNU's abstracted object file format. Quite often you have to check how symbols and logic map from the generic version to your actual target of interest. Sometimes the abstraction leaks, and object file specific code leaks in both the code and the definitions. Lastly, there is inevitably a lot of conversions from and to BFD.
- In general, the `tc-riscv.c` code gives me a great hint that it comes from a adapted stub of the MIPS code, which is okay considering their similarities, but some codepaths feel unnecessary at times.

With that said, I truly respect and admire the huge effort of a codebase like GNU `as` supporting basically every combination possible. My point is support all these combinations is not free, and you pay them in understanding overhead, build times, ease of hacking/refactoring, testing coverage, runtime performance, and overall care and attention you can pour into a piece of software.

This leads to what is an apple-to-orange comparison but `ras` is just above 10k lines of a single translation unit C project which, on my machine (M4 Pro), builds in ~200ms in debug mode (~300ms with sanitizers) using Clang, which is not exactly the fastest compiler, and ~600ms in release mode. By contrast, GNU `as` takes around 3s for a debug re-build, and a staggering ~45s in case the whole `configure` script is generated and runs. The latter contains ~20k lines of shell code generated by GNU `autoconf` and consisting of sequential checks needed to create the right build recipe.

### Readability and embedding

`ras` is unambiguously based on GNU `as`, streamlined: it strips away much of the generic-specific code tension, removes all global state usage, and re-interprets its core components. The result I've created is a much more readable version of a working assembler. I, for one, would have appreciated having something similar when starting. Another downstream effect of this is its a higher friendliness towards embedding or usage in other codebases: it is definitely not written with library usage in mind, because I don't have a precise idea of what it would mean for an assembler to be used as a library, however some of the primitives can be fairly easily copy-pasted and adapted to other needs.

## Challenges of writing an assembler

This has been my first time writing an assembler, and I would say that it came with a good set of challenges. Perhaps my very little background did not help: I've never written a lexer or an expression parser before, I didn't have much experience writing in C or in assembly, I didn't know much about ELF before, and I wasn't very comfortable in using debuggers. So at lot of things have been figured out for the first time. For this many reasons, of course I had to start with supporting only one architecture and object file format: I cannot reasonably expect to design a multi-arch/object assembler without knowing how to create a simpler one.

I also wanted to create something that _could be used_ in practice, and produce valid relocatable files. There is a _massive_ gap between a toy assembler, that is merely an instruction encoder, and an assembler that can be used in a compilation toolchain.

### Understanding it, as a whole

All of this is fine, as I was looking up for something new to learn! However, one particular point I've found struggling is that it is not really a "divide and conquer" problem: you cannot really work on subproblems A, B, C, do them individually, glue the code together and expect it to work. To write a decent implementation, or even understanding existing ones, you must have good context on how it works _as a whole_, how the various steps of assembling interact and how it plays with other programs that are part of the compilation toolchain.

More concretely, the major concepts: symbols, sections, fixups, relocations are all very intertwined and have to be understood all together across the different assembling steps and the object file format, otherwise you end up re-designing the same code many times because you stumble upon bad assumptions (what happened to me, indeed). One particular example is expression evaluation, where assessing what is means to perform an operation between symbols requires essentially all of the above and has different semantics depending on the assembly step you're in. [Here](https://github.com/thedevbirb/ras/blob/353e3939bf2dc892a55c92fcb87fde1e790f0d21/src/core/core_ir.c#L850-L902) is an example of the checks required to do such operations: while it may not seem a lot of code, you have to be aware of aforementioned concepts.

In particular reading either GNU `as` or LLVM MC can feel daunting. One of the reason is the size of the combinational space caused by different features, architectures and object file formats. This means reading has a lot of overhead because you have to do a lot of mental filtering of all the stuff you either don't need, or are not required for your target of interest, while trying to grasp the motivation for doing a certain operation in a larger context. Then, you have to pick your poison in terms of styling: GNU `as` features a distinctive '80s/'90s procedural C code, with too many globals (probably one hundred in total) and a lot of `#ifdef` checks to account; LLVM/LLVM MC is a typical object oriented codebase with a huge indirection due to classes, methods and virtual dispatch, so finding where stuff actually happens is non-trivial. Given I was writing C, and I wanted to follow a reference implementation, I've preferred GNU `as` over LLVM MC. I used the latter mainly for cross-checking some behaviour and to borrow some of its diagnostics design.

### Testing

Testing it's definitely one the hardest part required to make this software correct, and a weak point of my implementation. Testing throughly an assembler would probably require a large combination of different techniques, including fuzzing, golden testing, unit/integration testing, differential testing and so on. Bugs in an assembler, like in a compiler, can potentially lead to _executing unintended code_, and as such it must be taken with care. This excellent [talk](https://www.youtube.com/watch?v=V_qzqY1bb7I) by Richard Hipp, the primary author of SQLite, showcases how long and intensive can be testing software so that "it just works".

A pragmatic approach I've taken to reach a decent level of correctness is by differential testing with GNU `as` (commit `0a6aada`), using as source the SQLite library, amalgamed in a single C source file which compile to ~260k lines of assembly for bare-metal target. I've reached byte-per-byte equivalence on most sections of the object file, with the biggest difference remaining ordering of entries in the symbols table due to a different formatting approach. I acknowledge that this is by no means extensive and sufficient, but a decent tradeoff for _my goals_.

#### Digression on quirks

In reality, I'd prefer that testing an assembler would be much easier. However, I don't believe this is possible due to a strong lack of specifications of how an assembler should behave. While there is an RISC-V/ELF specification, in many cases it's not clear how some input is processed and treated. In GNU `as` some examples include:

1. A jump instruction with an expression containing one or more symbols behaves unreliably. Consider the source
   ```asm
   label1:
   label2:
   j label2-label1 + 8
   ```
   A consistent treatment of expressions with other parts of the codebase (as done by LLVM MC and `ras`) would fold such label different into a constant (0), and such instruction would be semantically equivalent to `j 8`. This the output of `readelf` instead:
   ```text
   Relocation section '.rela.text' at offset 0x1c8 contains 1 entry:
   Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
   0000000000000000  0000000500000011 R_RISCV_JAL            0000000000000000 label2 + 8
   ```
	Therefore this is translated into a completely wrong jump, without any warning emitted by GNU `as`. The reason for this behaviour is more subtle and it's due to the fixup data structure holding only a pointer to a single symbol, and not to an expression, to closely mimic the "relocation + addend" format of object files. However, this isn't necessary, you can have an expression pointer, and reduce the expression during fixup resolution.
2. No clear, centralised criteria on how symbols are kept in the symbols table. Symbols may be removed during assembly in some specific procedures, and some of them might be decided to be kept even if local and unused. For example, a file containing just `.local test` would place `test` in the symbols table as a globally undefined symbol, even if the directive is explicitly advertised to be used for local symbol creation. GNU `as` silently promotes locally undefined symbols to globals just before emission, even if the symbol is completely unused.  `ras` centralizes the decision making in a single [function](https://github.com/thedevbirb/ras/blob/40136a407b19fda99e5fc814713d9c3b79e78540/src/core/core_ir.c#L605-L644), however I'm pretty sure that I'm still missing edge cases and I still had to adapt its behaviour to ensure consistency with `as` output.
3. Some relocation operators should be rejected because their semantics would not apply, yet they are accepted, for example `lui x1, %pcrel_hi(my_label)`. `lui` is not a pc-relative instruction yet it accepts a pc-relative relocation with no complains from `as`. Luckily, LLVM MC rejects it with an appropriate error message; `ras` follows the same behaviour.
4. Without digressing further, the `.eqv` directive for forward-references expression has very non-trivial semantics especially with nested usage combined with the dot symbol.

There are many, many more subtle ones. To recap, there are some guardrails, but edge cases still slip in: the richness of the possible inputs combinations along with many directives having their own parsing rules make specifications, hence testing, difficult. As an implementer of an assembler, you then have to choose between either making your own rules or comply with what other assemblers, and therefore compilers, might expect. I acknowledge that both backwards compatibility and developing an assembler for a large number of hardware can contribute to this, again I'm not blaming the developers, but it's effectively a price to pay.

## A tour of a RISC-V assembler

I'll describe how an assembler works, making references to my implementation that can be found at https://github.com/thedevbirb/ras.

What is the role of an assembler? The task of an assembler is to ingest an input source written in a specific assembly language and generate _object files_, which is a structured binary representation of a program. This means it contains both the binary encoding of instructions and data blobs, along with metadata to tell other programs (most notably, a _linker_), how to use it. Most often, the input source is given from the compilation of a higher-level language, like C.

From the description above we can already infer that:

1. An object file must follow a specific format, which varies depending on the platform the code is intended to run (e.g. ELF for Linux and most embedded devices, PE/COFF on Windows, Mach-O for Apple operative systems etc). Other than outputting a different format, this has downstream effects on how the assembler works internally.
2. There isn't a specific assembly language, but a family of them. In particular, it varies between architecture (x86, ARM, RISC-V, etc), object file format and the assembler itself. Different assemblers might accept different syntaxes, while sharing a common core, although they target the same architecture and object format. Some assemblers offer very-rich macro-engines and constructs that can almost mimic a structured programming language like C. As such, the combinatorial space of possible options and configuration is very large.
3. The output of an assembler is not always something that you can run already. In particular, it can be a _relocatable object_, meaning that it contains symbol definitions that must be relocated by another program, like the linker.

After this preamble, I'll use a sort-of narrative approach to describe how an assembler works, explaining things as we go and starting from some compiler output, which is how most of the assembly is emitted nowadays nonetheless. Hand-writing assembly can still be extremely beneficial for performance sensitive code, where a programming language and its compiler may not capture as precisely the programmer's intent.

Let's compile an hello world program using GCC:

```c
#include <stdio.h>

int main()
{
    printf("hello from the ras assembler!\n");
    return 0;
}
```

In particular, we want to get rid of various debug (DWARF) and call frame information (CFI) which we won't be treated here, for a simpler assembly output. We get:

```asm
    .file      "main.c"
    .option    pic
    .attribute arch, "rv64i2p1_f2p2_d2p2_zicsr2p0"
    .attribute unaligned_access, 0
    .attribute stack_align, 16
    .text
    .section   .rodata
    .align     3
.LC0:
    .string   "hello from the ras assembler!"
    .text
    .align    2
    .globl    main
    .type     main, @function
main:
    addi    sp,sp,-16
    sd      ra,8(sp)
    sd      s0,0(sp)
    addi    s0,sp,16
    lla     a0, .LC0
    call    puts@plt
    li      a5,0
    mv      a0,a5
    ld      ra,8(sp)
    ld      s0,0(sp)
    addi    sp,sp,16
    jr      ra
    .size    main, .-main
    .section    .note.GNU-stack,"",@progbits
```

There is a lot to unpack! We will start to cover things as we go, doing deeper dives as needed.

### Lexing

First, we are dealing with ASCII text. As such, we first need some code that transforms raw text in _tokens_, which are a more structured representation of portions of text. For example, we might have tokens for identifiers, number literals, strings, mathematical operators and so on ([example](https://github.com/thedevbirb/ras/blob/40136a407b19fda99e5fc814713d9c3b79e78540/src/core/core_token.h#L6-L63)). Identifying portions of text via tokens help us _parse statements_: portions of text that the express some action that the assembler must do. Reading these tokens is done by a _lexer_ and called _lexing_ ([example](https://github.com/thedevbirb/ras/blob/40136a407b19fda99e5fc814713d9c3b79e78540/src/lexer.c#L42)). Note that indentation has no meaning in RISC-V assembly.

Here, there are already some design choices that one can make: should you lex the entire source code first, or do it as you go on demand, statement per statement? In practice, having a full list of tokens isn't very helpful and it can add unneeded memory usage: the assembler reasons in statements, so at least it would be more beneficial to have a full list of statements first, and reason entirely with this _intermediate representation_ (IR).

We can see many lines containing something that contain identifiers, in this case prefixed with a dot. Those are called _directives_. Syntactically, they differ from _labels_ which are identifiers ended by a colon (`:`). Directives, also called _pseudo-operations_ or _pseudo-opcodes_, are commands that can be given to an assembler to perform many different type of operations that may differ from encoding a specific instruction (unlike, let's say `addi sp,sp,-16`). There can be many directives: in case of GNU `as`, which is our reference implementation, there are over 100 of them https://www.sourceware.org/binutils/docs/as.html#SEC_Contents. Each directive has its own grammar and specific handling logic, which can also very between CPU architectures and object file formats. As mentioned previously, all this variety makes testing and writing a correct implementation pretty hard.

Listing some notable directives would be quite boring to read, and they require the very specific context they're used for. We will cover only a small portion of them in this post. Now, let's introduce one of the fundamental concepts of an assembler: sections and symbols.

### Sections

An object file is composed by sections, which contain binary data, that can happen to be instructions or follow a specific format. A _section header_ is placed in the _section header table_, mandatory for relocatable files, which specifies its content and structure. A section can be _empty_, but still be meaningful! [This picture](https://i.sstatic.net/RMV0g.png) can give a visual understanding on how they're laid out in an ELF file.
Most notable sections are: the `.text` section, containing executable data; the `.data` and `.rodata` sections containing static and static read-only data respectively; the `.bss` (block starting symbol) section _marks_ uninitialized static variables and has no size, the _program loader_ will take care of allocating such memory when loading the final linked program. The `.bss` section is the most notable example of a zero-sized section in a ELF file.
A compiler and an assembler may create many sections for different purposes, for example optimizations: dead code elimination and some linker optimizations rely on having one `.text`-like section _per function_. Other sections may include metadata that can be read in the object file, e.g. the last line of our source file containing `.section .note.GNU-stack,"",@progbits`. The `.section` directive is used to create them or _switch_ between them.
When assembling, sections can be thought as buffers of data: for example, writing instructions in the `.text` sections means writing blobs of possibly different sizes. While writing into sections, we might keep track of specifics offset within them. This is what _symbols_ are (mostly) for.

#### Quirks

The `.section` directive is complicated. On GNU `as` its parsing and semantics depends on a lot of factors, including:

1. the object file format used -- it accepts or not some extra arguments;
2. the name of the section provided -- if it has a [special name](https://gabi.xinuos.com/v42/elf/03-sheader.html#special-sections), some attributes are applied;
3. whether some additional arguments are provided and their value -- some combinations are optional, other mandatory;
4. whether the section is already defined -- if so, switch to it instead of creating it;
5. whether a _symbol_ with the same name has been defined and is overridable.

On a positive note, even the `as` [manual](https://www.sourceware.org/binutils/docs/as.html#g_t_002esection-name) for the directive gives already an hint on how delicate that is. Nonetheless, when you're implementing it feels hard to get right. Even while writing this note I suspected about some edge case I wasn't handling very well and I had to squash a couple of bugs. Sure, I won't deny some skill issue on my side, but I also feel part of the design should have been different. _It's not something that should be hard in the first place_.
### Symbols

Before, we've mentioned _labels_ as identifiers with a `:` placed after them:

```asm
# ...
.LC0:
    .string    "hello from the ras assembler!"
# ...
main:
    addi    sp,sp,-16
# ...
```

Such syntax creates "labels", which are a type of symbols that represents offsets within a section. For example, when reading the object file produced by an assembler of our source code using `readelf`, we can notice that `.LC0` has a _value_ of zero, meaning that it starts at offset zero in its section, in this case the `.data` section. If there were another label in the data section, let's say `.LC1`, right after the `.string` directive, its offset would be the size of the null-terminated string `"hello from the ras assembler!"`.
I'm using the term _offset_ because during assembly there is no real notion of an _address_. An address will be set by the linker or the program loader afterwards, adding to such offsets a base address. As such, section offsets represented by symbols indicate actual address only when the program is loaded. In a higher level programming language, `.LC0` represents a pointer to a static null-terminated string.

One of the first subtleties is that sections are (almost always) symbols, and they appear in the symbols table: they're named after themselves, and their value is always zero in a relocatable file, and they constitute a valid jump target. So `j .text + 8` is a valid instruction that tells the CPU to jump (modify the program counter) to the address of the `.text` section, and then add 8 bytes as offset. The symbol table is itself a section, called `.symtab`, however for such section there is no entry in the symbols table. The same applies for the `.strtab` and `.shstrtab` which contain the strings that constitute the name of symbols and sections, respectively.

Here is a view from `readelf` of the symbols table for our program:

```text
Symbol table '.symtab' contains 12 entries:
   Num:    Value          Size Type    Bind   Vis+Other Ndx(SecName) Name [+ Version Info]
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 SECTION LOCAL  DEFAULT    1 (.text)   .text
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    3 (.data)   .data
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    4 (.bss)    .bss
     4: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS           main.c
     5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5 (.rodata) .rodata
     6: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT    5 (.rodata) .LC0
     7: 0000000000000010     0 NOTYPE  LOCAL  DEFAULT    1 (.text)   .L0
     8: 0000000000000000     0 SECTION LOCAL  DEFAULT    6 (.note.GNU-stack) .note.GNU-stack
     9: 0000000000000000     0 SECTION LOCAL  DEFAULT    7 (.riscv.attributes) .riscv.attributes
    10: 0000000000000000    56 FUNC    GLOBAL DEFAULT    1 (.text)   main
    11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND           puts
```

#### Absolute and undefined symbols

There are other ways to declare symbols, most notably using the directives `.set` or `.equ`. The syntax is `.set <name>, <expression>` or `<name> = <expression>` (this one is not supported in `ras`). Such directives create a symbol with attached an expression, that can be evaluated. If the expressions can be resolved to a constant, the symbol will belong to the _absolute section_, which has no size and marks constants, where the value doesn't represent an offset but rather the evaluation of its expression.

An expression, either inside a `.set` directive or inside an instruction, can contain the name of a symbol which hasn't been defined in a source file. In the example provided, that is the case for `puts` inside `call puts`. `puts` will be resolved during linking as a `libc` symbol for a `printf`-like function which takes no formatting arguments. Such symbol during assembly is simply not known at all, and it will be associated to the _undefined_ section, another zero-sized section used to mark symbols which are unknown.

In particular, the `.set` directive creates _volatile_ symbols, which can be redefined. However, previous reference _must remain available_, and only the last one will end up in the symbols table. In contrast, the `.equiv` directive creates non-volatile symbols. This has implications in the way you define the data structure holding symbols: for example, a regular hashmap wouldn't work unless your values are pointers to symbols. Contiguity of data is also very important for iteration performance: `malloc`-ing each symbol separately can result in scattered memory, a reason why a dedicated allocator can be better for this purpose.

In the `readelf` dump above, the only absolute (`ABS`) symbol we're seeing is `main.c`, created by the `.file` directive that marks a symbol type as `FILE`. Such symbol is defined and must belong to some section, and it has been chosen for it to the absolute section. Why? I don't know, probably to avoid creating a specific one for it and pollute tables.

#### The dot symbol

There is one special symbol, called _dot_, that appears in source files as `.`. In our example, it appears in the line `.size main, . - main`. The value of the dot in a _non-forward_ expression (e.g. `.set`) is the current offset within the section, also called "location counter". Since `main` appears at offset zero in the `.text` section in our example, and there are `0x56` (86) bytes of instructions before the `.size` directive, the expression resolves to `0x56` and the symbol has a size of 86 bytes. As you may have guessed, the `.size` directive sets the size of a symbol; it's useful metadata for _disassembling_ and debugging, since it establish where a function body ends.
In practice, every usage of a dot symbol inside the an assembler source code is converted into an internal label, with a name like `.L0 `: as such, the dot symbol is continuously updated as sections are written, and on every usage a snapshot of it is created and marked with an internal name.

#### Operations between symbols

The line `.size main, . - main` expresses a subtraction between two labels. Such expression is well-defined and has meaning in the context of an assembler. However, `. + main` and most of the other binary operators do not have a valid semantics and are not allowed. Back to subtraction between symbols, there are still a [lot of checks to be made before it can be considered valid](https://github.com/thedevbirb/ras/blob/b1f0ff1656b15922b8f637ab77289d2e6a4b5815/src/core/core_ir.c#L879-L904), and the result can be simplified into a constant. It's perfectly fine to not understand it at first, since there are also concepts I haven't explained yet and cannot fit  in this already long post; this is to show an example of a very common, primitive operation happening in an assembly source file that requires almost the entire context of how an assembler works to implement correctly!
To my knowledge, such operations aren't specified, so to understand what compilers and people expect you have to dive into a source code and just make sense of it after some effort.
#### Quirks of `as`

You would expect that a label definition, a `.section` directive and `.set` directive would all accept the same grammar for names, since they all create entries in the symbols table. That's not the case, and now consider the line `.section .note.GNU-stack`: `.section` accepts `.` and `-`, a label definition rejects `-`, a `.set` directive rejects both. However, you can use any ASCII sequence, including escape sequences, if you write the name between double quotes (`"`); unluckily the `"name" = <expression>` syntax is not accepted.

It is possible to write `.set ., 1234` and assigning it a value, or create a `.` label with no complains by `as` (at least by default). In such case, a `.` identifier will appear in the symbols table. However, it cannot actually be used in any expression, because `.` will be replaced with the location counter.
### Instructions

Instructions represents the third and last type of statements we can have in RISC-V/GNU assembly syntax, along with directives and label definitions. Therefore, anything that doesn't start with a `.` and doesn't end with a `:` is tried to be parsed as an instruction. In our example, everything after `main` apart the bottom directives are instructions:

```asm
addi    sp,sp,-16
sd      ra,8(sp)
sd      s0,0(sp)
addi    s0,sp,16
lla     a0,.LC0
call    puts@plt
li      a5,0
mv      a0,a5
ld      ra,8(sp)
ld      s0,0(sp)
addi    sp,sp,16
jr      ra
```

Parsing an instruction results in the assembler encoding bytes in an executable section of the object file, such as `.text`. You may see that they accept variable number of arguments of different kinds: each instruction may belong to a certain "enconding" family and share the same syntax, while there are others which are aliases (e.g. `mv`) or small macros that expand to multiple instructions (e.g. `call`, `li`).

The variety in the encodings is somewhat annoying to deal with, but in practice a good table design makes the work mostly straightforward. The approach here is to define an instruction as a series of atomic pieces that should be parsed, and then a generalized parser that reads such atoms from a [table](https://github.com/thedevbirb/ras/blob/2e5de45439d60dd62e14cc4e0318fc9b04f1b876/src/riscv/riscv_instruction.c#L41-L415). As such, in my experience encoding instructions has been the easiest part of doing the assembler, and now an LLM can assist you very well in this typo-friendly workload which is taking an instruction from the specification and write you an entry in your table.

What is way more complex to handle about instructions (and some directives) are two properties they may exhibit:

1. the expressions they contain might not be resolvable;
2. they can shrink or grow in size.

To represents these possibilities, we need to introduce other core components of an assembler: fixups, relocations, fragments and relaxation.
#### Fixups and relocations

The instruction `call` is an example of both. Its semantics is indeed one of the function call: jump where a certain symbol (label) is defined, and then come back where you left. In this case, the expression of `call` evaluates to the symbol `puts` (ignore the `@plt`) that hasn't been defined before, and we don't know its value. It might be defined later, or not at all like in our case.
How can we encode a jump-like instruction if we don't know which bits to write as jump destination? We don't, and this what relocations are for. From `readelf` output of the object file `main.o`, we can read the following relocation entries:

```text
Relocation section '.rela.text' at offset 0x78 contains 6 entries:
    Offset             Info             Type               Symbol's Value  Symbol's Name + Addend
0000000000000010  0000000600000017 R_RISCV_PCREL_HI20     0000000000000000 .LC0 + 0
0000000000000010  0000000000000033 R_RISCV_RELAX                             0
0000000000000014  0000000700000018 R_RISCV_PCREL_LO12_I   0000000000000010 .L0  + 0
0000000000000014  0000000000000033 R_RISCV_RELAX                             0
0000000000000018  0000000b00000013 R_RISCV_CALL_PLT       0000000000000000 puts + 0
                                                                             ^--- that's our boi!
0000000000000018  0000000000000033 R_RISCV_RELAX                             0
```

The ELF object file format has special sections containing relocations, named after existing sections, e.g. `.rela.text` here. `rela` stands for "`rel`ocation with `a`ddend" since on the right we can see a symbol name and an addend that acts as an offset.
The goal of a relocation is to notify to the a consumer of this file, like the linker, that some information is missing, and must be provided elsewhere before the program can executed as intended.
The `Offset` field indicates where in the target section (`.text`) the "patch" must be applied. The `Info` field contains both the index of the required symbols table entry (`0xb`), and the relocation type (`0x13`, which translates to `R_RISCV_CALL_PLT`) that specifies which bits to modify to patch the instruction.

Relocations are the format accepted in the object file to encode these patches. Internally, an assembler keeps a list of "fixups", which are a richer representation of pending relocations that might be resolved or not. If resolved, they are dropped, otherwise they appear in the object file.

Fixups are a mainly a prerogative of single pass assembler. Such design imposes to read the source text only once, and to not iterate over an intermediate representation of the whole code. Therefore, when an unknown symbol is found a fixup is created internally even if the same symbol might be defined shortly after. There will be a step in the assembly that iterates over the set of fixups and tries to resolve them.
Multi-pass assembler might scan the whole source code first and track all symbols defined in it, however there is still some accounting to be made for those which end up not being defined, so in practice there isn't much of a difference.

#### Fragments and relaxation

I've mentioned that the `call` instruction can shrink or grow in size. What does it mean, precisely? See, a jump destination is encoded as some _signed distance from the program counter_ (PC-relative distance), with tells the CPU the next instruction to fetch from memory. Let's consider this simpler example to convey such idea:

```asm
beq t0, t1, my_function
# a lot of other code
my_function:
```

The `beq` opcode stands for "branch on equal" and jump to the label contained in the expression on the right if the two registers (in this case, `t0` and `t1`) contain the same bits. It is the analogous of something like `if (a == b) { my_function() }` in other languages. Such instructions features 12 bits to encode a distance to jump to; since an odd address is invalid, it is assumed the first bit is zero, so it expresses the range `[-4096, 4096)` instead of `[-2048, 2048)`.
What if there is enough code between `beq` and `my_function` that is over 4KiB? One option would be to error, and explicitly asking the programmer to do this instead:

```asm
bne t0, t1, +8
j my_function
# a lot of other code
my_function:
```

That is, negate the condition and ask to jump past the `j` instruction. The latter has 21 bits to use i.e., a `[-2MiB, 2MiB)` range, to express a distance. But now, we've added an instruction, thus shifting all the offset of all others as well. In practice, instead of erroring, an assembler would implicitly convert the original `beq` into the snippet above.

More generally, some instructions like (un)conditional jumps and some directives have to expand due their relationship with other parts of the code. At the same time, we want to shrink them whenever possible to avoid executing extra instructions, costing both unnecessary cycles and bytes. Sometimes you really can't know the distance because the symbols is undefined, e.g. with `call puts`. In fact, `call` is a small macro that expands to two instructions that allows to jump to 32 bits of distance. The `R_RISCV_RELAX` relocation notifies the linker that it is possible to shrink this operation to just one instruction if it close enough.

A first challenge during assembly is therefore starting to populate a section contents knowing that some instructions or directives can leave some wiggle room. This is what "fragments" are needed for: they represents ordered buffers of the encoded content of a section, with an optional variable length tail to accommodate special instructions and directives. Concretely, instructions are written into fragments and labels are associated to them using also an offset for precise location.
After all the source text has been processed we need to to create the object file, and we want to glue together all fragments to create the actual section content, and try to shrink the size of the variable tails as such as possible. This is where [_relaxation_](https://github.com/thedevbirb/ras/blob/b1f0ff1656b15922b8f637ab77289d2e6a4b5815/src/core/core_ir.c#L1895-L2125) comes in: a monotonic convergence algorithm that starts by assuming the worst variable size possible, and continuously iterate over fragments to shrink them (if possible) until we find a fixed point, where we cannot shrink further.
#### Some words on RISC-V encodings and extensions

Instructions can have different encodings, meaning that their arguments, like registers and immediates, can be arranged differently depending on their size. For example, in RISC-V, general-purpose registers are 32 and occupy 5 bits each, but you may restrict their usage to 16 of them and use 1 bit less in some instructions; similarly, immediates can range from 12 bits to 20. Instructions are usually 32 bits long, and optional 16 bits instructions are supported via an extension.

This variety results in six different encodings (R, I, S, B, U, J) used across the _base set of instructions_. To complicate a bit, the RISC-V ISA is made by too many extensions of the base instruction set `I`, which is of little use alone. There are 11 "standard" extensions (not counting `I`), and then a separate collection over 100 of them, prefixed with a "Z". By default, every extension is optional, but some extension may implicitly activate or require others, and some of them are incompatible. If you want to know in your code which extensions are available on  the CPU running it, the latter must support the "Control-Status Register" `Zicsr` extension, which is again optional, and apparently the specification allows the `misa` register to optionally read as `zero`, providing no further information. This is explained in great detail [here](https://dmitry.gr/?r=06.%20Thoughts&proj=12.%20RV).
Extensions have also the consequence to naturally introduce new encodings. In theory, this is fine, but sometimes there can be too many. The compressed instruction extension `C`, which introduces 16 bits instruction to reduce code density, adds encodings that apply to only one instruction at a time.
Lastly, all this variety and optionality results in some level of overhead and boilerplate, other than hardware complexity. From an assembler point of view, GNU `as` alone dedicates [3k lines of code to extensions tables, parsing and validation logic](https://github.com/gnutools/binutils-gdb/blob/09a7362522ef0447210042a4f0309559edae82bb/bfd/elfxx-riscv.c#L1157-L4325), with more of it necessary inside instruction parsing. For a comparison, the same lines of code are required in `ras` to write most of its core data structures and handlers.

## Design considerations

Overall, I'd say the assembly syntax seems a higher-level language/pseudocode that targets a specific architecture/object file, rather than being formal and low-level. The reason in my opinion could be mostly historical and due to a lot of hand-writing assembly before compilers where invented, so _maybe_ (I don't know such history yet, pardon me) there wasn't a strong clue of what worked or not in terms of syntaxes, and most of them were result of trial and error. Backwards compatibility also further enshrined some patterns, for example LLVM MC (which started its development in 2009) maintains most of the parity with GNU syntax and directives.

Some tension in assemblers design could be due to the fact that they're used by both compilers and humans, which are clearly a different audience, landing on a design that is not ideal for both. I have no experience in compiler development, but to me it seems that assembly text is not an ideal low-level IR and there is complexity inherited by it. GCC must produce assembly text and pipe it into GNU `as` directly, while LLVM-based compilers can skip this step calling the MC API; however the latter is written to support both compiler and hand-written assembly text. On the opposite side, as a human you've a language that, outside of basic instruction writing, lacks accurate specification and validations, with 100+ keywords (directives) that have their own parsing rules and non-trivial semantics. At the same time, it is not sufficiently low-level because it abstracts over the object file emission; the relationship between the syntax and the resulting object file is sometimes not straightforward, and may require knowing assembler internals to understand some quirks that might happen.

Maybe a better approach, even if radical, would be to give up entirely on the assembly text, and use typed APIs and hygienic macros in programming languages to manipulate/inspect object files and instruction encoding, similar to how JIT assemblers/compilers work. Inline assembly support is already a compile-time feature of some programming languages, and such APIs could be exposed and executed during such step. [Odin `asm` templates](https://github.com/odin-lang/Odin/pull/7271) are an interesting example in a similar direction that I've seen recently.

## Implementation

Here I'll talk about parts of the `ras` implementation, describing some of the design choices, what worked well for me and my experience.
### Memory and data structures

This codebase uses _only_ arena allocators (here is an [excellent resource](https://www.dgtlgrove.com/p/untangling-lifetimes-the-arena-allocator) for reading more about them), which have proven to be very easy to use and simple to implement in just 300 [lines of C]([https://github.com/thedevbirb/ras/blob/main/src/base/base_arena.c](https://github.com/thedevbirb/ras/blob/b1f0ff1656b15922b8f637ab77289d2e6a4b5815/src/base/base_arena.c)). In general, an assembler that runs as a single thread, short lived binary doesn't require that much attention to memory allocation: Data is continuously allocated as more information is collected from the source, and is almost never freed until the very end after writing the object file. So a naive strategy of just `malloc`-ing and let the OS free memory at teardown of the process is perfectly acceptable. Despite this, arenas allow more granural control over contiguous memory compared to an interface like `malloc`, which is key for modelling _fragments_.
#### Fragments

Previously we've briefly defined fragments as ordered, contiguous buffers within a section. In `ras`, every section has a list of [`Fragment`s](https://github.com/thedevbirb/ras/blob/b1f0ff1656b15922b8f637ab77289d2e6a4b5815/src/core/core_ir.h#L448-L481) with a dedicated arena allocator that is used for this sole purpose. This unlocks several important properties:
- Data can be appended to fragments as needed, without committing upfront to a certain size (apart the minimum of a OS page).
- Fragments are laid almost all contiguously in memory, for fast iteration (important for the relaxation algorithm) and individual fragment data is guaranteed to be contiguous.
- There is never a necessity to re-allocate memory, as such you can assume _stable pointers_ and reliably take references to data knowing it won't become stale due to re-allocations.
It is the ideal memory allocation strategy for the ideal data structure to model the problem of dynamically sized instructions and directives in assemblers. Traditional memory allocation interfaces would either result in more scattered memory or more complex APIs. That is a beauty, and not a burden, of manual memory management.

#### The symbols table, and the rest

Along with fragments, the [symbols table](https://github.com/thedevbirb/ras/blob/b1f0ff1656b15922b8f637ab77289d2e6a4b5815/src/core/core_ir.h#L273-L297) represent the second most important data structure inside `ras`. It's implement as a hash trie backed by a singly-linked, doubly-ended list (a queue). Elements within the list are allocated using a dedicated arena. At the time of writing this post, it is used to also allocate expressions, sections and fixups, since they all share the same lifetime (yes, some interfaces should change to reflect that, I know, but I realized it later). This design is friendly towards a wide variety of usages of such structure:

1. Symbols are allocated in memory close to each other, useful since it necessary to iterate over all of them in the final steps of assembling.
2. Symbols can be redefined (see `.set`), and for name lookup only the most recent definition is wanted. Redefined symbols remain in the same underlying list.
3. No re-allocation and therefore no-rehashing is ever needed.
4. It is safe to take references to symbols, simplifying a lot some APIs and skipping unnecessary lookups. This is again because of stable pointers guaranteed by the arena and because the table _never_ re-allocates.

In general, linked lists work great paired with arena allocators (see another [excellent article](https://www.dgtlgrove.com/p/in-defense-of-linked-lists) by Ryan Fleury on the topic). If serial dependency of data access is a real concern, you can still use chunk lists i.e., a linked list of array chunks, as a knob to tune it while retaining all their flexibility and guaranteed stable pointers.

[This file](https://github.com/thedevbirb/ras/blob/b1f0ff1656b15922b8f637ab77289d2e6a4b5815/src/core/core_ir.h#L448-L481) contains the internal representation of many concepts of an assembler that we have discussed so far: expressions, symbols, sections, fragments, fixups, and so on. All of them rely on the same heuristics and techniques mentioned previously. The results are very powerful data structures and a succinct 3k lines of code implementation (including comments and wide formatting) of the core of the assembler. While it may not be perfect and for sure contains some errors, I think there is value in doing so much with so little.

### Diagnostics

`ras` features rich, location based diagnostics to pinpoint errors precisely. For this feature I inspected LLVM codebase because I remembered that `clang` has very good diagnostics. The essence of it is in virtual locations: all inputs source are mapped into a virtual offset space referred by diagnostics. Example of it in action, in case of semantically invalid usage of relocation operators:

```shell
ras ./examples/test.o -o ./examples/test.o
./examples/test.s:2:9: error[E05005]: invalid relocation operator for instruction
    2 | lui x1, %pcrel_hi(my_label)
      | ~~~     ^~~~~~~~~
```

Expression can also feature very rich diagnostics. At the cost of 12 extra bytes (three `U32`s) in the `Expression` struct, you can have precise tracking like the following:

```shell
ras ./examples/test.o -o ./examples/test.o
./examples/test.s:3:44: error[E03005]: expression cannot be fully resolved and finalized
    3 | .set TEST, (label_1 - label_2) + (global_1 - global_2)
      |                                  ~~~~~~~~~~^~~~~~~~~~~
```

That is, `ras` is able to pinpoint the exact sub-expression and the operation (`-`) that can't be performed on those two symbols. It could be even more precise and accurate; this is a work in progress. However, there is a powerful engine backing the diagnostics of this assembler.

Last showcase consists of symbol re-definition:

```shell
ras ./examples/test.o -o ./examples/test.o
./examples/test.s:6:6: error[E06001]: symbol cannot be redefined
    6 | .set TEST, 5
      |      ^~~~
./examples/test.s:1:6: note: previous declaration is here
    1 | .eqv TEST, 1
      |      ^~~~
```

### On writing C

As I mentioned at the beginning of this post, I wanted to write it in C with no dependencies, forcing me to do and choose everything from scratch. I was inspired by this after reading the [RADDebugger](https://github.com/EpicGames/raddebugger) codebase, which remains an endless source of inspiration on how to write C proficiently, without sacrificing ergonomics, and while being extremely readable and grep-able.
The assembler has been a perfect project for this goal, since there isn't much that you need to get started: its [base layer](https://github.com/thedevbirb/ras/tree/main/src/base) contains mainly an allocator, string utilities, and some mathematical helpers, mostly for bitwise operations and alignment. Careful usage of the C preprocessor macros goes a long way: you can define your own keywords over compiler attributes, safer iteration for common data structures, scope-based hooks and so on. I invite you to read the base layer of the RADDBG codebase for more details.
It's still some work to do, I acknowledge that, but in practice once you write your base layer, with what you like, you can bring it over to the new projects that you build, and it evolves with you; the return on investment is fairly high!
#### Know your hardware and compiler!

While C is a ISO standardized language, there are a lot of details about how it should be implemented that are either _unspecified_ or _undefined_. There are countless discussions on the internet about this topic, since there are nasty bugs that can happen due to this. One of my favourite resource on the topic is this [blog](https://faultlore.com/blah/c-isnt-a-language/) along with every reference written there. C is full of quirks, historical baggage, bad implementations, errors etc, mostly due its large number of implementations (it's way easier to do things when there is just one!). The C standard is an high-level guardrail on parts of the language that implementers must agree on, or have the freedom to adapt. The core language is very small however, with the C99/C11 standard being ~150 pages of text excluding standard libraries, which should be used very sparingly anyway.

However, those "footguns" can be greatly reduced by not adhering to the fantasy that you should write software only for the [_C abstracted machine_](https://www.dependablec.org/#As-If). Code gets compiled with specific compilers and runs on specific hardware. Writing code without any (checked) assumption on either your hardware or compiler that you're using leaves a lot of ergonomics and safety on the table. This ranges from how you manage memory to helpers, warnings, etc. In particular, warnings can greatly shape how you write the language itself. You don't like variable shadowing? Disable it via compiler flags. Same for a lot of type conversions and promotions that normally would go silent. You can decide to crash on integer overflow/underflow, to use compiler-specific saturated arithmetic and so on. A C standard is only a shared interface to which compilers adapt and you can rely on.

### Building

For building `ras` you only need a C compiler. There are no separate tools, and not even shell scripts of which I always forget the syntax (yes, skill issue). Building recipes are made with the fantastic [nob.h](https://github.com/tsoding/nob.h) single-header file library made by Tsoding, so you keep writing only in C and you have all the features of the programming language you're using to build your software. Other languages that natively marry this approach are [Jai](https://jaiprogramming.com/) and [Zig](https://ziglang.org/learn/build-system/).

## Next steps and final considerations

This project took almost 5 months to reach this state. I have been slow, but so many things were figured out on the first time. In practice, understanding and designing took most of the time. A lesson learnt is definitely to spend more time ahead to minimize rewrites, and re-evaluating your understanding. Sometimes I felt unproductive for this, but productiveness is a very weird goal to chase and very nuanced, highly dependent on your _goals_.

Some extra time was due to almost dropping it in a couple of occasions because yeah, what the hell am I doing with this? I don't know. I've already mentioned what is the value I've found in this project, yet I don't know whether there will be a practical need from other people that I work on it. Perhaps I'll continue improving it as a side project and use it to assemble some code on whatever microcontroller I might want to play with (a CH32V003 is nice starter for complete bare-metal), and dive deeper in the world of understanding hardware.

At the same time, I feel some architecture-specific toolchain could be much, much better than existing offerings. I don't believe adapting the same gigantic codebases over and over to support new architectures can achieve something as good and lean as writing it for this architecture only. In order for RISC-V to win more market share over time, the software close to the hardware must be as good as the hardware itself. Both x86 and ARM have more decades of efforts poured into software support; RISC-V in comparison is still pretty young.
I think it because while this is my first attempt at writing an assembler, with very little background and with some months of effort, I've done something interesting although not truly great. It's interesting because in one try I've achieved some appreciable properties not shared by other assemblers. By truly great, I mean an order of magnitude better than existing offerings, where switching to using it is a no-brainer. I'm okay with it, given it's my first attempt, and I share the [idea](https://x.com/antovsky/status/2089204943521796547) that you have to write the same thing multiple times in order to fully understand it and make something great. I want to work on toolchain software, understand hardware more and make RISC-V great. Please, contact me if that appeals to you!

On a separate note, I'd like to start writing more posts like this, and maybe some programming videos. A classic could be one about not using recursion at all for expression parsing or evaluation, as [I've done in this codebase](https://github.com/thedevbirb/ras/blob/b1f0ff1656b15922b8f637ab77289d2e6a4b5815/src/core/core_ir.c#L732), which has been both fun and actually helpful to understand such algorithms better.

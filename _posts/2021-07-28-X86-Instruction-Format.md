---
title: X86 Instruction Format
categories:
  - blog
tags:
  - X86
  - ISA
---
# Learn by yourself
* [Intel Software Developer’s Manual](https://software.intel.com/content/www/us/en/develop/articles/intel-sdm.html)
    - The most authoritative and official reference manual
    - Volume 2 out of 4 volumes: Instruction Set Reference A-Z  
* [sandpile.org](https://www.sandpile.org/index.htm)
    - The world's leading source for technical x86 processor information
    - Compact and summarized tables are provided. (for expert)
* [XED](https://intelxed.github.io/)
    - The X86 Encoder Decoder (used by [Pin](https://software.intel.com/en-us/articles/pin-a-dynamic-binary-instrumentation-tool) and the [Intel® Software Development Emulator (SDE)](https://software.intel.com/en-us/articles/intel-software-development-emulator))
* [QEMU](https://github.com/qemu/qemu)  
    - A generic and open source machine & userspace emulator and virtualizer.
* [LLVM](https://github.com/llvm/llvm-project)
    - X86Instr*.td,  X86MCCodeEmitter.cpp

# Instruction Format Overview
![overview](/rc/X86-Instruction-Format/instruction-format.png)

# ModR/M
![modrm](/rc/X86-Instruction-Format/modrm.png)

Many instructions that refer to an operand in memory have an addressing-form specifier byte (called the ModR/M byte) following the primary opcode. The **ModR/M** byte contains three fields of information:

- The **mod** field combines with the r/m field to form 32 possible values: eight registers and 24 addressing modes
- The **reg/opcode** field specifies either a register number or three more bits of opcode information. The purpose of the reg/opcode field is specified in the primary opcode.
- The **r/m** field can specify a register as an operand or it can be combined with the mod field to encode an addressing mode. Sometimes, certain combinations of the mod field and the r/m field are used to express opcode information for some instructions.

# SIB bytes
![sib](/rc/X86-Instruction-Format/sib.png)

Certain encodings of the ModR/M byte require a second addressing byte (the **SIB** byte). The base-plus-index and scale-plus-index forms of 32-bit addressing require the SIB byte. The SIB byte includes the following fields:

The scale field specifies the scale factor.
The index field specifies the register number of the index register.
The base field specifies the register number of the base register.

# Displacement and Immediate
Some addressing forms include a displacement immediately following the ModR/M byte (or the SIB byte if one is present). If a displacement is required, it can be 1, 2, or 4 bytes.If an instruction specifies an immediate operand, the operand always follows any displacement bytes. An immediate operand can be 1, 2 or 4 bytes.

In 64bit mode, the typical size of immediate operands remains 32 bits. When the operand size is 64 bits, the processor **sign-extends** all immediates to 64 bits prior to their use. MOV is the only instruction that has 64bit immediate.

![imm](/rc/X86-Instruction-Format/imm.png)

(3) and (4) are same,  (2) is not encodable

# SDM Instruction Representation
![mov-1](/rc/X86-Instruction-Format/mov-1.png)
![mov-2](/rc/X86-Instruction-Format/mov-2.png)

```console
echo ‘movl $1024, 16(%eax,%ebx,4)’ | llvm-mc --triple=i386 --show-encoding
xed -e 'MOV MEM4:EAX,EBX,4,10 IMM:00004000'
```

* `c7` => 0xc7
* `/0` => Reg: 000
* `SIB+disp8` => Mod:01 R/M:100
* `eax` => Base:000
* `ebx*4` => SS:10 Index:011
* `16(disp8)` => 0x10
* `$1024(i32)` => 0x400 (0x00 0x40 0x00 0x00)

Final encoding:

Op   | ModRM              | SIB                 | Disp | Immediate
---  | ---                | ---                 | ---  | ---
0xc7 | 0x44(0b01'000'100) | 0x98(0b10'011’000)  | 0x10 | 0x00 0x40 0x00 0x00  

# XED Instruction Representation
https://intelxed.github.io/ref-manual/group__IFORM.html  (xed/datafiles)

* XED uses domain-specific mini-language to specify the formats of all x86 instructions (IFORM).
* Components are separated by underline "_".  The 1st component is the instruction name (ADD, SUB, SHL, etc). 
* After that there is one component for each operand, starting from the destination.
* GPR means general purpose register，MEM means memory, IMM means immediate, etc.  In x86 encoding there are typically an 8-bit version and a variable-osize (16b or 32b or 64b) version of each instruction type.  That explains the last letter "8" or "v".
* IMMz means that the immediate is 16b when osize is 16b and 32b when osize is 32b or 64b. 

```console
$ xed -d 'c7 44 98 10 0040 00 00'  
C744981000400000                                  
ICLASS:     MOV                                   
CATEGORY:   DATAXFER                              
EXTENSION:  BASE                                  
IFORM:      MOV_MEMv_IMMz                         
ISA_SET:    I86                                   
ATTRIBUTES: HLE_REL_ABLE SCALABLE                 
SHORT:      mov dword ptr [eax+ebx*4+0x10], 0x4000
```

```console
$ xed -d '83 c3 01'
83C301                            
ICLASS:     ADD                   
CATEGORY:   BINARY                
EXTENSION:  BASE                  
IFORM:      ADD_GPRv_IMMb         
ISA_SET:    I86                   
ATTRIBUTES: SCALABLE              
SHORT:      add ebx, 0x1 
```

```console
$ xed -d '80 c3 01'
80C301                            
ICLASS:     ADD                   
CATEGORY:   BINARY                
EXTENSION:  BASE                  
IFORM:      ADD_GPR8_IMMb   
ISA_SET:    I86                   
ATTRIBUTES: BYTEOP                
SHORT:      add bl, 0x1 
```

# Legacy prefix
OSZ (0x66): The operand size override prefix is used to switch the operand data size from 32b to 16b, 16b to 32b. The OSZ prefix is also used to provide one more opcode bit in many cases, without affecting the operand data size.

ASZ (0x67): The address size override prefix. Outside of 64b mode, the ASZ prefix generally switches 32b memory addressing to 16b addressing or vice versa. In 64b mode, the ASZ prefix switches operations to 32b memory addressing away from the default of 64b addressing.

```console
$ echo 'addw $1, %bx' | llvm-mc --triple=i386 --show-encoding
addw    $1, %bx                # encoding: [0x66,0x83,0xc3,0x01]

$ echo 'addl $1, %ebx' | llvm-mc --triple=i386 --show-encoding
addl    $1, %ebx               # encoding: [0x83,0xc3,0x01]
```

```console
$ echo 'movq $1024, 16(%rax,%rbx,4)' | llvm-mc --triple=x86_64 --show-encoding  
movq    $1024, 16(%rax,%rbx,4)  # encoding: [0x48,0xc7,0x44,0x98,0x10,0x00,0x04,0x00,0x00]

$ echo 'movq $1024, 16(%eax,%ebx,4)' | llvm-mc --triple=x86_64 --show-encoding
movq    $1024, 16(%eax,%ebx,4) # encoding: [0x67,0x48,0xc7,0x44,0x98,0x10,0x00,0x04,0x00,0x00]
```

# Opcode
A primary opcode can be 1, 2, or 3 bytes in length. An additional 3-bit opcode field is sometimes encoded in the ModR/M byte. Smaller fields can be defined within the primary opcode. Such fields define the direction of operation, size of displacements, register encoding, condition codes, or sign extension. Encoding fields used by an opcode vary depending on the class of operation.

Two-byte opcode formats for general-purpose and SIMD instructions consist of one of the following:
An **escape opcode byte** 0FH as the primary opcode and a second opcode byte.
A **mandatory prefix** (66H, F2H, or F3H), an escape opcode byte, and a second opcode byte (same as previous bullet).

For example, CVTDQ2PD consists of the following sequence: F3 0F E6. The first byte is a mandatory prefix (it is not considered as a repeat prefix).

Three-byte opcode formats for general-purpose and SIMD instructions consist of one of the following:
An escape opcode byte 0FH as the primary opcode, plus two additional opcode bytes.
A mandatory prefix (66H, F2H, or F3H), an escape opcode byte, plus two additional opcode bytes (same as previous bullet).

For example, PHADDW for XMM registers consists of the following sequence: 66 0F 38 01. The first byte is the mandatory prefix.

In the legacy space, we have currently populated 4 maps. 
Map 0 includes the 1-byte opcodes and escapes for accessing other maps.
Map 1 opcodes are prefaced by a 0x0F escape.
Map 2 opcodes are prefaced by a 0x0F 0x38 escape.
Map 3 opcodes are prefaced by an 0x0F 0x3A escape.

# REX prefix (0x40-0x4F)
![rex-1](/rc/X86-Instruction-Format/rex-1.png)
![rex-2](/rc/X86-Instruction-Format/rex-2.png)

Note: Besides W bit,  the LSB of opcode is used to distinguish the operand size.

```console
$ echo 'addb $1, %bl' | llvm-mc --triple=i386 --show-encoding
addb    $1, %bl                # encoding: [0x80,0xc3,0x01]
$ echo 'addw $1, %bx' | llvm-mc --triple=i386 --show-encoding
addw    $1, %bx                # encoding: [0x66,0x83,0xc3,0x01]
$ echo 'addl $1, %ebx' | llvm-mc --triple=i386 --show-encoding
addl    $1, %ebx               # encoding: [0x83,0xc3,0x01]
$ echo 'addq $1, %rbx' | llvm-mc --triple=x86_64 --show-encoding
addq    $1, %rbx               # encoding: [0x48,0x83,0xc3,0x01]
```
IA-32 mode : 
	* 0x40 - 04f  single-byte INC/DEC instructions
64-bit mode: 
	* 0x40 - 0x4f REX prefix
	* 0xff /0, 0xff /1 INC/DEC

# Redundant encoding
## Reverse encoding
![add](/rc/X86-Instruction-Format/add.png)

R/M can encodes a register or memory operand

* Add r32 to r/m32
* Add r/m32 to r32     <----- Reversed encoding in LLVM

## Disp
![disp8-disp32](/rc/X86-Instruction-Format/disp8-disp32.png)

* 0 can be encodes as a disp8; 
* disp8 can be encoded as disp32

## RIP-relative addressing
![rip-1](/rc/X86-Instruction-Format/rip-1.png)
![rip-2](/rc/X86-Instruction-Format/rip-2.png)
![rip-3](/rc/X86-Instruction-Format/rip-3.png)

A new addressing form, RIP-relative (relative instruction-pointer) addressing, is implemented in 64-bit mode. An effective address is formed by adding displacement to the 64-bit RIP of the next instruction.

In IA-32 architecture and compatibility mode, addressing relative to the instruction pointer is available only with control-transfer instructions. In 64-bit mode, instructions that use ModR/M addressing can use RIP-relative addressing. Without RIP-relative addressing, all ModR/M modes address memory relative to zero.

# Encoding limits
![limit-1](/rc/X86-Instruction-Format/limit-1.png)
![limit-2](/rc/X86-Instruction-Format/limit-2.png)

* RSP based memory must be encoded w/ SIB
* RSP can not be used as index register
* RIP-relative addressing ignores REX prefix. In order to address R13 with no displacement, software must encode R13 + 0 using a 1-byte displacement of zero.

```console
$ echo 'movq (%rip), %rdx' | llvm-mc --triple=x86_64 --show-encoding
movq    (%rip), %rdx  # encoding: [0x48,0x8b,0x15,0x00,0x00,0x00,0x00]
$ echo 'movq (%r13), %rdx' | llvm-mc --triple=x86_64 --show-encoding
movq    (%r13), %rdx  # encoding: [0x49,0x8b,0x55,0x00]
```
Difference
*  0x48 <-> 0x49 REX.B
*  0x15 <-> 0x55  (mod=00, reg=010, r/m=101  <-> mod=01, reg=010, r/m = 101) 

## EFLAGS
https://www.sandpile.org/x86/flags.htm

![eflags-1](/rc/X86-Instruction-Format/eflags-1.png)
![eflags-2](/rc/X86-Instruction-Format/eflags-2.png)
![eflags-3](/rc/X86-Instruction-Format/eflags-3.png)
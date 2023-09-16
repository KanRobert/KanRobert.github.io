---
title: All about APX relocation
categories:
  - RFC
tags:
  - APX
  - X86
---
# Reference
1. https://www.intel.com/content/www/us/en/developer/articles/technical/advanced-performance-extensions-apx.html
1. https://maskray.me/blog/2021-02-14-all-about-thread-local-storage
1. https://gitlab.com/x86-psABIs/x86-64-ABI

# REX2
Intel® APX introduces a new 2-byte REX2 prefix in order to support 32 general-purpose registers (GPRs).

The **REX2** prefix is two bytes long and consists of a byte of value 0xD5 followed by a second byte containing payload bits. The payload bits `[W,R3,X3,B3]` have the same meanings as the REX payload bits `[W,R,X,B]`, except that the instructions `PUSH reg` with opcodes 0x50-0x57 and `POP reg` with opcodes 0x58-0x5F in legacy map 0 will use `REX2.W` to encode the `PPX` (push-pop acceleration) hint (that's another story). The payload bits `[R4,X4,B4]` provides the fifth and most significant bits of the `R`, `X` and `B` register identifiers, each of which can now address all 32 GPRs. Like the **REX** prefix, when OSIZE = 8b, the presence of the REX2 prefix makes GPR ids [4,5,6,7] address byte registers `[SPL,BPL,SIL,DIL]`, instead of `[AH,CH,DH,BH]`.

A REX2-prefixed instruction is always interpreted as an instruction in legacy map 0 or 1, with `REX2.M0` encoding the map id. REX2 does not support any instruction in legacy maps 2 and 3. Intel® APX extension of legacy instructions in maps 2 and 3 (such as CRC32 and SHA instructions) is provided by the extendedEVEX prefix.

REX2 | Payload byte 1
---  | ---
0xD5 | M0 R4 X4 B4 W R3 X3 B3

# `R_X86_64_REX2_GOTPCRELX` or `R_X86_64_CODE_4_GOTPCRELX`
In a previous [article](https://kanrobert.github.io/blog/X86-relocation), we introduced relocation `R_X86_64_GOTPCRELX`, `R_X86_64_REX_GOTPCRELX` and GOT optimization. Linker needs to know how many bytes to look back from the relocated place to precisely determine which instruction we're dealing with for GOT optimzation.

![overview](/rc/X86-Instruction-Format/instruction-format.png)

The assembly representation of GOT relocation is

```
# a.s
movq   foo@GOTPCREL(%rip), %rax
```

From the perspective of instruction encoding, the relative address of symbol `foo` to register `RIP` is encoded in field `Displacement` and there is no SIB. So, `GOTPCRELX` looks back two bytes, `REX_GOTPCRELX` three and now if we define `REX2_GOTPCRELX`, it looks back four bytes.

In case we have other 2-byte prefix in the future, HJ (the maintainer of x86 abi) proposed the name `R_X86_64_CODE_4_GOTPCRELX` [here](https://groups.google.com/g/x86-64-abi/c/KbzaNHRB6QU).

Jan Beulich had a more aggressive proposal to use the relocated field to hold the offset to the start of the instruction (or the major opcode), so only one kind of relocation `R_X86_64_CODE_X_GOTPCRELX` (or `R_X86_64_OPCODE_GOTPCRELX`) would suffice for all prefixes, even for possible `VEX`(2/3 bytes) or `EVEX`(4 bytes) relocations in the future.

Howerver, currently the assembler encodes zeros at the relocation offset and RELA relocation ignores the value. The proposal `R_X86_64_OPCODE_GOTPCRELX` would break the compatibilty and was rejected.

```console
$ cat a.s | as  -o a.o
$ objdump -d a.o

a.o:     file format elf64-x86-64


Disassembly of section .text:

0000000000000000 <.text>:
   0:   48 8b 05 00 00 00 00    mov    0x0(%rip),%rax        # 0x7
```

Same as `R_X86_64_GOTPCRELX`/`R_X86_64_REX_GOTPCRELX`, relocation [`R_X86_64_CODE_4_GOTPCRELX`](https://groups.google.com/g/x86-64-abi/c/saQyqBeL5XE) applies to instructions

```
mov name@GOTPCREL(%rip), %reg
test %reg, name@GOTPCREL(%rip)
binop name@GOTPCREL(%rip), %reg
```

where binop is one of `adc, add, add, cmp, or, sbb, sub, xor` instructions. We define the enumeration value of this relocation as 43

```c
# define R_X86_64_CODE_4_GOTPCRELX 43
```

If the instruction starts at 4 bytes before the relocation offset. It is
similar to `R_X86_64_GOTPCRELX`. Linker can treat `R_X86_64_CODE_4_GOTPCRELX`
as `R_X86_64_GOTPCREL` or convert the above instructions to

```
lea name(%rip), %reg
mov $name, %reg
test $name, %reg
binop $name, %reg
```

if the first byte of the instruction at the relocation `offset - 4` is `0xd5`
when possible.

# Thread-local storage
Thread-local storage (TLS) provides a mechanism allocating distinct objects for different threads. It is the usual implementation for GCC extension `__thread`, C11 `_Thread_local`, and `C++11 thread_local`, which allow the use of the declared name to refer to the entity associated with the current thread. You can find detailed introduction at [All about thread local storage](https://maskray.me/blog/2021-02-14-all-about-thread-local-storage) and [ELF Handling for Thread-Local Stroage](http://www.akkadia.org/drepper/tls.pdf)


# `R_X86_64_CODE_4_GOTTPOFF`
For initial exec TLS model (executable & preemptible), the offset from the thread pointer to the start of a static block is fixed at program start, such an offset can be encoded by a GOT relocation. For x86-64, the relocation is `R_X86_64_GOTTPOFF`, which can used by TLS instructions

```
add name@gottpoff(%rip), %reg
mov name@gottpoff(%rip), %reg
```

If the offset can be determined at link time, linker can convert the above instructions to

```
add	$name@tpoff, %reg
mov	$name@tpoff, %reg
```

For APX, we add

```c
#define R_X86_64_CODE_4_GOTTPOFF 44
```

for instruction starts at 4 bytes before the relocation offset. This should be used if reg is one of the additional general-purpose registers, r16-r31, in Intel APX. It is similar to `R_X86_64_GOTTPOFF` and linker optimization must take the different instruction encoding into account.

# TLS descriptors
Some architectures (arm, aarch64, i386, x86-64) have TLS descriptors as more efficient alternatives to the traditional general dynamic and local dynamic TLS models. Such ABIs repurpose the first word of the (module ID, offset from dtv[m] to the symbol) pair to represent a function pointer. The function pointer points to a very simple function in the static TLS case and a function similar to `__tls_get_addr` in the dynamic TLS case. The caller does an indirection function call instead of calling `__tls_get_addr`. There are two main points:

* The function call to `__tls_get_addr` uses the regular calling convention: the compiler has to make the pessimistic assumption that all volatile registers may be clobbered by `__tls_get_addr`
* In glibc (which does lazy TLS allocation), `__tls_get_addr` is very complex. If the TLS of the module is backed by a static TLS block, the dynamic loader can simply place the TP offset into the second word and let the function pointer point to a function which simply returns the second word.

The first point is the prominent reason that TLS descriptors are generally more efficient. Loading an int32_t using x86-64 TLSDESC has a code sequence like the following

```
leaq    x@TLSDESC(%rip), %rax
call    *x@TLSCALL(%rax)
movl    %fs:(%rax), %eax
```

where `x@TLSDESC(%rip)` is of type `R_X86_64_GOTPC32_TLSDESC`. When the code executes, the call instruction will call the `__tlsdesc_static` function, loading the TP offset from the second word. Then the third instruction `movl %fs:(%rax), %eax` performs a `%fs` relative memory load.

# R_X86_64_CODE_4_GOTPC32_TLSDESC
Linker can convert the `leaq    x@TLSDESC(%rip), %rax` to

```
mov	$name@tpoff, %reg
mov	name@gottpoff(%rip), %reg
```

if possible. See https://reviews.llvm.org/D114416 for reference. To put it simple, **`TLSDESC` calls a function to calculate the TP OFFSET, `gottpoff` loads TP offset from GOT entry. tpoff is the TP offset**.

For APX, we add

```c
#define R_X86_64_CODE_4_GOTPC32_TLSDESC 45
```

for instruction starts at 4 bytes before the relocation offset. This should be used if reg is one of the additional general-purpose registers, r16-r31, in Intel APX. It is similar to `R_X86_64_GOTPC32_TLSDESC` and linker optimization must take the different instruction encoding into account.
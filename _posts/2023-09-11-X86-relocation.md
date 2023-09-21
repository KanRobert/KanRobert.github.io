---
title: X86 relocation
categories:
  - blog
tags:
  - X86
  - ABI
---

# What's relocation
Relocations are entries in binaries that are left to be filled later -- at link time by the toolchain linker or at runtime by the dynamic linker. A relocation in a binary is a descriptor which essentially says "determine the value of `X`, and put that value into the binary at offset `Y`" — each relocation has a specific type, defined in the ABI documentation, which describes exactly how "determine the value of" is actually determined.

Here `X` is the address of a symbol (e.g. a function or variable), and may be:
* a link-time constant
* the load base plus a link-time constant
* depedent on runtime computation by ld.so

and `Y` is a compile-time constant. So we can classify relocation by the kinds of symbol address. 

# Kinds of relocation
## Link-time constant
For this case, this component(an executable or shared object) must be an position-dependent executable: a link-time address equals its virtual address at run-time.
### `R_X86_64_32S`, `R_X86_64_32`, `R_386_32`
```
$ cat a.s
.text
    mov foo, %eax
```
```console
$ as a.s -o a.o
$ readelf -r a.o

Relocation section '.rela.text' at offset 0xc8 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000003  00040000000b R_X86_64_32S      0000000000000000 foo + 0
```
The value of foo is not known at the time you make a.o, so the compiler leaves behind a relocation (of type `R_X86_64_32S`) which is saying "in the final binary, patch the value at offset 0x3 in this object file with the address of `foo + 0`. If you take a look at the output, you can see at offset 0x3 there are 4-bytes of zeros just waiting for a real address:
$ objdump -d a.o
```console
0000000000000000 <.text>:
   0:   8b 04 25 00 00 00 00    mov    0x0,%eax
```
When we change target to i386 from x86-64, the relocation become `R_386_32`
```console
$ as --32 a.s -o a.o
$ readelf -r a.o

Relocation section '.rel.text' at offset 0x94 contains 1 entry:
 Offset     Info    Type            Sym.Value  Sym. Name
00000001  00000401 R_386_32          00000000   foo

$ objdump -d c.o
00000000 <.text>:
   0:   a1 00 00 00 00          mov    0x0,%eax
```
The suffix `S` in `R_X86_64_32S` is short for sign-extend, which is introduced in x86-64 b/c the type of 32-bit immediate of 64-bit instructions is signed instead of unsigned, e.g. the semantic of `addq %rax, 0xff112233` is `rax <- rax + sign_extend(0xff112233)`.

x86-64 also supports unsigned relocation, e.g.
``` console
$ cat b.s
.text
    .long foo

$ as b.s -o b.o
$ readelf -r b.o

Relocation section '.rela.text' at offset 0xc8 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000000  00040000000a R_X86_64_32       0000000000000000 foo + 0
```
The difference between `R_X86_64_32S` vs `R_X86_64_32` is when the linker will complain "with relocation truncated to fit":

* `32`: complains if the truncated after relocation value does not zero extend the old value, i.e. the truncated bytes must be zero. E.g.: FF FF FF FF 80 00 00 00 to 80 00 00 00 generates a complaint because FF FF FF FF is not zero.

* `32S`: complains if the truncated after relocation value does not sign extend the old value. E.g.: FF FF FF FF 80 00 00 00 to 80 00 00 00 is fine, because the last bit of 80 00 00 00 and the truncated bits are all 1.

### How to read the relocation entry
The Info column caters two purposes
* gives the index of the symbol in the symbol table
* gives the type of relocation

```c
#define ELF32_R_SYM(x) ((x) >> 8)
#define ELF64_R_SYM(i)                  ((i) >> 32)

#define ELF32_R_TYPE(x) ((x) & 0xff)
#define ELF64_R_TYPE(i)                 ((i) & 0xffffffff)
```
`00040000000a = 4 << 32 + 10`, the symbol `foo` can be found t the index 4 of `.symtab`, and the corresponding relocation type can found in psABI (Processor Specific Application Binary Interface) documentation

```console
$ readelf -s b.o

Symbol table '.symtab' contains 5 entries:
   Num:    Value          Size Type    Bind   Vis      Ndx Name
     0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
     1: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
     2: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
     3: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
     4: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND foo
```

Name | Value | Field | Calculation
---  | ---   | ---   | ---
R_X86_64_NONE | 0 | none | none
R_X86_64_64 | 1 | word64 | S + A
R_X86_64_PC32 | 2 | word32 | S + A - P
R_X86_64_GOT32 | 3 | word32 | G + A
R_X86_64_PLT32 | 4 | word32 | L + A - P
R_X86_64_COPY | 5 | none | none
R_X86_64_GLOB_DAT | 6 | wordclass | S
R_X86_64_JUMP_SLOT | 7 | wordclass | S
R_X86_64_RELATIVE | 8 | wordclass | B + A
R_X86_64_GOTPCREL | 9 | word32 | G + GOT + A - P
R_X86_64_32 | 10 | word32 | S + A
R_X86_64_32S | 11 | word32 | S + A
R_X86_64_16 | 12 | word16 | S + A
R_X86_64_PC16 | 13 | word16 | S + A - P
R_X86_64_8 | 14 | word8 | S + A
R_X86_64_PC8 | 15 | word8 | S + A - P
R_X86_64_DTPMOD64 | 16 | word64 |
R_X86_64_DTPOFF64 | 17 | word64 |
R_X86_64_TPOFF64 | 18 | word64 |
R_X86_64_TLSGD | 19 | word32 |
R_X86_64_TLSLD | 20 | word32 |
R_X86_64_DTPOFF32 | 21 | word32 |
R_X86_64_GOTTPOFF | 22 | word32 |
R_X86_64_TPOFF32 | 23 | word32 |
R_X86_64_PC64 | 24 | word64 | S + A - P
R_X86_64_GOTOFF64 | 25 | word64 | S + A - GOT
R_X86_64_GOTPC32 | 26 | word32 | GOT + A - P
R_X86_64_SIZE32 | 32 | word32 | Z + A
R_X86_64_SIZE64 | 33 | word64 | Z + A
R_X86_64_GOTPC32_TLSDESC | 34 | word32 |
R_X86_64_TLSDESC_CALL | 35 | none |
R_X86_64_TLSDESC | 36 | word64 × 2 |
R_X86_64_IRELATIVE | 37 | wordclass | indirect (B + A)
R_X86_64_RELATIVE64 | 38 | word64 | B + A
Deprecated | 39
Deprecated | 40
R_X86_64_GOTPCRELX | 41 | word32 | G + GOT + A - P
R_X86_64_REX_GOTPCRELX | 42 | word32 | G + GOT + A - P


where
* `A` Represents the addend used to compute the value of the relocatable field.
* `B` Represents the base address at which a shared object has been loaded into memory
during execution. Generally, a shared object is built with a 0 base virtual address,
but the execution address will be different.
* `G` Represents the offset into the global offset table at which the relocation entry’s symbol
will reside during execution.
* `GOT` Represents the address of the global offset table.
* `L` Represents the place (section offset or address) of the Procedure Linkage Table entry
for a symbol.
* `P` Represents the place (section offset or address) of the storage unit being relocated (computed using r_offset).
* `S` Represents the value of the symbol whose index resides in the relocation entry.
* `Z` Represents the size of the symbol whose index resides in the relocation entry.

We will give an example below to calculate `R_X86_64_PC32`.
## Load base plus constant
For this case, this component is either a position-independent executable or a shared object. The difference between the link-time addresses of two symbols equals their virtual address difference at run-time. The first byte of the program image, the ELF header, is loaded at the load base. The text section can obtain the current program counter, then add the distance from the PC to the symbol (PC-relative address), to compute the run-time virtual address.
### R_X86_64_PC32
```c
// c.c
extern int foo;
int bar() { return foo; }
```
```console
bash$ gcc -c c.c
bash$ readelf -a c.o
Relocation section '.rela.text' at offset 0x198 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000000006  000900000002 R_X86_64_PC32     0000000000000000 foo - 4

bash$ objdump -d c.o

0000000000000000 <bar>:
   0:   55                      push   %rbp
   1:   48 89 e5                mov    %rsp,%rbp
   4:   8b 05 00 00 00 00       mov    0x0(%rip),%eax        # a <bar+0xa>
   a:   5d                      pop    %rbp
   b:   c3                      retq
```
At link time,  the relocation will go away and filled by `S + A - P`, where
* `S` = the address of foo = 0x404028
* `A` = -4
* `P` = offset of the relocation = 0x40110c
, and result is `0x2f18`

 ```c
// d.c
int foo = 0xab;
int bar();
int main() { return bar(); }
```
```console
$ gcc -c d.c -o d.o
$ gcc c.o d.o -o a.out
$ readelf -r a.out | grep "foo"

$ objdump -d a.out
0000000000401106 <bar>:
  401106:       55                      push   %rbp
  401107:       48 89 e5                mov    %rsp,%rbp
  40110a:       8b 05 18 2f 00 00       mov    0x2f18(%rip),%eax        # 404028 <foo>
  401110:       5d                      pop    %rbp
  401111:       c3                      retq

0000000000401112 <main>:
  401112:       55                      push   %rbp
  401113:       48 89 e5                mov    %rsp,%rbp
  401116:       b8 00 00 00 00          mov    $0x0,%eax
  40111b:       e8 e6 ff ff ff          callq  401106 <bar>
  401120:       5d                      pop    %rbp
  401121:       c3                      retq

$ readelf -s a.out | grep "foo"
    85: 0000000000404028     4 OBJECT  GLOBAL DEFAULT   22 foo
```
## Runtime computation by ld.so
For this case, we need help from the runtime loader (abbreviated as ld.so). The linker emits a dynamic relocation to let the runtime loader perform a symbol lookup to determine the associated symbol value at runtime.
### Why dynamic relocation
The major reason, as I shall try to explain, is position-independent code (PIC). When you look at an executable file, you'll notice it has a fixed load address
```console
$ readelf -l a.out

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000001000 0x0000000000401000 0x0000000000401000
                 0x0000000000000131 0x0000000000000131  R E    0x1000
  LOAD           0x0000000000002e40 0x0000000000403e40 0x0000000000403e40
                 0x00000000000001ec 0x00000000000001f0  RW     0x1000
```
This is not position-independent. The code section (with permissions R E; i.e. read and execute) must be loaded at virtual address 0x401000, and the data section (RW) must be loaded above that at exactly 0x403e40.

This is fine for an executable, because each time you start a new process (fork and exec) you have your own fresh address space. Thus it is a considerable time saving to pre-calculate addresses from and have them fixed in the final output.

This is not fine for a shared library (.so). Shared libraries have two goals
* applications pick-and-choose random permutations of libraries to achieve what they want.
* code sharing. If a hundred processes use a shared library and the code is completely read-only, then only one copy is needed in the memory every process can share the same code.

 If your shared library is built to only work when loaded at one particular address everything may be fine until another library comes along that was built also using that address. The problem is actually somewhat tractable — you can just enumerate every single shared library on the system and assign them all unique address ranges, ensuring that whatever combinations of library are loaded they never overlap. This is essentially what prelinking does (although that is a hint, rather than a fixed, required address base). But that would be a maintenance nightmare.

 Also if you patch the code and inform it where to find the data, it would destroy the read-only property of the code and thus sharability.

```console
$ gcc -fPIC -shared -o libc.so c.c
$ readelf -l libc.so

Program Headers:
  Type           Offset             VirtAddr           PhysAddr

  LOAD           0x0000000000001000 0x0000000000001000 0x0000000000001000
                 0x0000000000000115 0x0000000000000115  R E    0x1000
  LOAD           0x0000000000002e28 0x0000000000003e28 0x0000000000003e28
                 0x00000000000001f8 0x0000000000000200  RW     0x1000
```
The column `VirtAddr` for a shared library only mean an offset instead of an abosulte virtual address.

### How to implement dynamic relocation
#### Global offset table
 If a shared library can be loaded at any address, then how does an executable, or other shared library, know how to access data or call functions in it? As we know, all problems can be solved with a layer of indirection, in this case called **Global Offset Table** (abbreviated as GOT). The compiler emits code which uses position-independent addressing to extract the absolute virtual address from GOT. The relocations (e.g, `R_X86_64_GOTPCRELX`, `R_X86_64_REX_GOTPCRELX`) are called GOT-generating. The linker will create entries in the Global Offset Table.
 
 The Global Offset Table (usually consists of `.got` and `.got.plt`) holds the symbol addresses which are referenced by text sections. `.got.plt` holds symbol addresses used by PLT entries. `.got` holds everything else.

#### `R_X86_64_GOTPCRELX`, `R_X86_64_REX_GOTPCRELX`
```console
$ gcc -fPIC -shared -S -o - c.c

bar:
...
    movq    foo@GOTPCREL(%rip), %rax # R_X86_64_REX_GOTPCRELX
    movl    (%rax), %eax
...
```
In assembly, both of them are represnted as `foo@GOTPCREL(%rip)`, where `foo` is the symbol name.

```console
$ objdump -d libc.so
00000000000010f9 <bar>:
    10f9:       55                      push   %rbp
    10fa:       48 89 e5                mov    %rsp,%rbp
    10fd:       48 8b 05 e4 2e 00 00    mov    0x2ee4(%rip),%rax        # 3fe8 <foo>
    1104:       8b 00                   mov    (%rax),%eax
    1106:       5d                      pop    %rbp
    1107:       c3                      retq

$ readelf -S libc.so
Section Headers:
  [Nr] Name              Type             Address           Offset
       Size              EntSize          Flags  Link  Info  Align
  ...
  [18] .got              PROGBITS         0000000000003fd8  00002fd8
       0000000000000028  0000000000000008  WA       0     0     8
  ...

$ readelf -r libc.so

Relocation section '.rela.dyn' at offset 0x3a8 contains 8 entries:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000003fe8  000300000006 R_X86_64_GLOB_DAT 0000000000000000 foo + 0
```
The disassembly shows that the value to be returned is loaded from an offset of 0x2ee4 from the current %rip; i.e. 0x3fe8. Looking at the section headers, we see that this is part of the .got section. 

When we examine the relocations, we see a `R_X86_64_GLOB_DAT` relocation that says "find the value of symbol foo and put it into address 0x3fe8.

So, when this library is loaded, the dynamic loader will examine the relocation, go and find the value of foo and patch the .got entry as required. When it comes time for the code loads to load that value, it will point to the right place and everything just works; without having to modify any code values and thus destroy code sharability.

#### `R_X86_64_PLT32`
This handles data, but what about function calls? The indirection used here is called a procedure linkage table or PLT. Code does not call an external function directly, but only via a PLT stub.

```c
// e.c
int foo();
int bar() { return foo(); }
```

```console
$ gcc -fPIC -shared -S -o - e.c

bar:
...
    call    foo@PLT
...
```
In assembly, it's represnted as `foo@PLT`, where `foo` is the symbol name.
```console
$ gcc -fPIC -shared -o libe.so e.c
$ objdump -d libe.so
0000000000001030 <foo@plt>:
    1030:       ff 25 e2 2f 00 00       jmpq   *0x2fe2(%rip)        # 4018 <foo>,  The first call
    1036:       68 00 00 00 00          pushq  $0x0                 #    jumps here
    103b:       e9 e0 ff ff ff          jmpq   1020 <.plt>

0000000000001109 <bar>:
    1109:       55                      push   %rbp
    110a:       48 89 e5                mov    %rsp,%rbp
    110d:       b8 00 00 00 00          mov    $0x0,%eax
    1112:       e8 19 ff ff ff          callq  1030 <foo@plt>
    1117:       5d                      pop    %rbp
    1118:       c3                      retq

$ readelf -r libe.so

Relocation section '.rela.plt' at offset 0x450 contains 1 entry:
  Offset          Info           Type           Sym. Value    Sym. Name + Addend
000000004018  000300000007 R_X86_64_JUMP_SLO 0000000000000000 foo + 0
```
we can see that function makes a call via `foo@plt`. It's an interesting call, we jump to the value stored in 0x2fe2 past the current `%rip` (i.e. 0x4018), which we can then see the relocation for — the symbol `foo`.

It is interesting to keep following this through; let's look at the initial value that is jumped to:
```console
$ objdump -D --start-address=0x4018 --stop-address=0x4020 libe.so
Disassembly of section .got.plt:

0000000000004018 <_GLOBAL_OFFSET_TABLE_+0x18>:
    4018:       36 10 00                adc    %al,%ss:(%rax)
    401b:       00 00                   add    %al,(%rax)
    401d:       00 00                   add    %al,(%rax)
```
Unscrambling 0x4018 we see its initial value is 0x1036 — i.e. the next instruction! Then it pushes the value 0 and jumps to 0x1020. Looking at that code we can see it pushes a value from the GOT, and then jumps to a second value in the GOT:
```console
Disassembly of section .plt:

0000000000001020 <.plt>:
    1020:       ff 35 e2 2f 00 00       pushq  0x2fe2(%rip)  # 4008 <_GLOBAL_OFFSET_TABLE_+0x8>, Push the descriptor
    1026:       ff 25 e4 2f 00 00       jmpq   *0x2fe4(%rip) # 4010 <_GLOBAL_OFFSET_TABLE_+0x10>, Jump to the PLT resolver in ld.so
    102c:       0f 1f 40 00             nopl   0x0(%rax)
```
What's going on here? What's actually happening is lazy binding (deferered bing). In the lazy binding scheme, ld.so does minimum work to ensure the jmp instruction lands in a trampoline which will tail call the PLT resolver in ld.so. The `pushq` instructions pushes a descriptor (for the `.got.plt` entry) and jumps to the PLT header. The PLT header is a stub calling the function address stored at `.got.plt+0x10` (PLT resolver in ld.so) with an extra argument (descriptor for the current component).

### GOT optimization
When the symbol associated to a GOT entry is [non-preemptible](https://a.atmos.washington.edu/~ovens/ifort90_doc/main_for/mergedProjects/optaps_for/fortran/optaps_cmp_visib_f.htm), the third case effectively becomes the first or the second case. The code sequence nevertheless has a load from the GOT entry. Why don't we optimize the code sequence?

For example, x86-64's `R_X86_64_GOTPCRELX` and `R_X86_64_REX_GOTPCRELX` optimization can transform a load/store via GOT to a direct load/store, and a GOT-indirect call/jump to a direct call/jump.
```
# input
movq    var@GOTPCREL(%rip), %rax  # R_X86_64_REX_GOTPCRELX
movl    (%rax), %eax

callq   *func@GOTPCREL(%rip)      # R_X86_64_GOTPCRELX

jmpq    *func@GOTPCREL(%rip)      # R_X86_64_GOTPCRELX

# output
leaq    var(%rip), %rax
movl    (%rax), %eax

addr32 callq func

jmpq    func
nop
```

# Reference
1. https://maskray.me/blog/2021-08-29-all-about-global-offset-table
1. https://maskray.me/blog/2021-09-19-all-about-procedure-linkage-table
1. https://www.technovelty.org/linux/plt-and-got-the-key-to-code-sharing-and-dynamic-libraries.html
1. https://gitlab.com/x86-psABIs/x86-64-ABI
1. https://stackoverflow.com/questions/6093547/what-do-r-x86-64-32s-and-r-x86-64-64-relocation-mean
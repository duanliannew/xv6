Inspect stack content in gdb, try to figure out what the content is.
```
(gdb) x/24x $esp
0x7bdc:	0x00007d87 # bootmain calls kernel entry, this is the return address of calling kernel entry
	0x00000000	0x00000000	0x00000000
0x7bec:	0x00000000	0x00000000	0x00000000	0x00000000
0x7bfc:	0x00007c4d # return address for calling function bootmain
	0x8ec031fa # code for symbol start
        0x8ec08ed8	0xa864e4d0
0x7c0c:	0xb0fa7502	0xe464e6d1	0x7502a864	0xe6dfb0fa
0x7c1c:	0x16010f60	0x200f7c78	0xc88366c0	0xc0220f01
0x7c2c:	0x087c31ea	0x10b86600	0x8ed88e00	0x66d08ec0

```
Why are there 28 consecutive bytes of 0s?
Check the prolog of bootmain, there are 16 bytes of push (ebp, edi, esi, ebx), 16 bytes of stack expanding, 12 bytes of push (0x0, 0x1000, 0x10000), 16 bytes of stack shrinking, so there are 28 bytes of push in total, and they are all 0s.

```
   0x7d3d:	push   %ebp
   0x7d3e:	mov    %esp,%ebp
   0x7d40:	push   %edi
   0x7d41:	push   %esi
   0x7d42:	push   %ebx
   0x7d43:	sub    $0x10,%esp
   0x7d46:	push   $0x0
   0x7d48:	push   $0x1000
   0x7d4d:	push   $0x10000
   0x7d52:	call   0x7cf4
   0x7d57:	add    $0x10,%esp
```
0. Check the boot process
```
bootasm.s:start --> start32 --> bootmain --> entry()
```

1. Read the bootblock part of the Makefile, we find that the entry symbol is "start" in bootasm.S, which is designated adress 0x7c00.
   Verify: use nm to inspect symbols, which outputs below.
```
        nm bootblock.o
        00007d3d T bootmain
        00007e74 R __bss_start
        00007e74 R _edata
        00007e74 R _end
        00007c60 t gdt
        00007c78 t gdtdesc
        00007c8c T readsect
        00007cf4 T readseg
        00007c09 t seta20.1
        00007c13 t seta20.2
        00007c5c t spin
        00007c00 T start
        00007c31 t start32
        00007c7e T waitdisk
```

2. Read code in boostasm.s, we find that it only setup stack inside start32 (as noted from previous output: 0x7c31).
```
.code32  # Tell assembler to generate 32-bit code now.
start32:
  # Set up the protected-mode data segment registers
  movw    $(SEG_KDATA<<3), %ax    # Our data segment selector
  movw    %ax, %ds                # -> DS: Data Segment
  movw    %ax, %es                # -> ES: Extra Segment
  movw    %ax, %ss                # -> SS: Stack Segment
  movw    $0, %ax                 # Zero segments not ready for use
  movw    %ax, %fs                # -> FS
  movw    %ax, %gs                # -> GS

  # Set up the stack pointer and call into C.
  movl    $start, %esp
  call    bootmain

  # If bootmain returns (it shouldn't), trigger a Bochs
  # breakpoint if running under Bochs, then loop.
  movw    $0x8a00, %ax            # 0x8a00 -> port 0x8a00
  .....
```

3. Inspect instruction address of start32
```
(gdb) x/15i 0x7c31
   0x7c31:	mov    $0x10,%ax
   0x7c35:	mov    %eax,%ds
   0x7c37:	mov    %eax,%es
   0x7c39:	mov    %eax,%ss
   0x7c3b:	mov    $0x0,%ax
   0x7c3f:	mov    %eax,%fs
   0x7c41:	mov    %eax,%gs
   0x7c43:	mov    $0x7c00,%esp
   0x7c48:	call   0x7d3d
   0x7c4d:	mov    $0x8a00,%ax
   0x7c51:	mov    %ax,%dx
   0x7c54:	out    %ax,(%dx)
   0x7c56:	mov    $0x8ae0,%ax
   0x7c5a:	out    %ax,(%dx)
   0x7c5c:	jmp    0x7c5c
```

4. Inspect instruction of bootmain (0x7d3d)
```
(gdb) x/100i 0x7d3d
   0x7d3d:	push   %ebp
   0x7d3e:	mov    %esp,%ebp
   0x7d40:	push   %edi
   0x7d41:	push   %esi
   0x7d42:	push   %ebx
   0x7d43:	sub    $0x10,%esp
   0x7d46:	push   $0x0
   0x7d48:	push   $0x1000
   0x7d4d:	push   $0x10000
   0x7d52:	call   0x7cf4
   0x7d57:	add    $0x10,%esp
   0x7d5a:	cmpl   $0x464c457f,0x10000
   0x7d64:	jne    0x7d87
   0x7d66:	mov    0x1001c,%eax
   0x7d6b:	lea    0x10000(%eax),%ebx
   0x7d71:	movzwl 0x1002c,%esi
   0x7d78:	shl    $0x5,%esi
   0x7d7b:	add    %ebx,%esi
   0x7d7d:	cmp    %esi,%ebx
   0x7d7f:	jb     0x7d96
   0x7d81:	call   *0x10018 # kernel entry
   0x7d87:	lea    -0xc(%ebp),%esp
   ...
```

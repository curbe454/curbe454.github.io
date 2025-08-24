---
title: "Ocaml_reverse_start"
description: 
date: 2025-06-02T04:21:25+08:00
image: 
math: 
license: 
hidden: false
comments: true
draft: false
categories:
    - reverse
tags:
    - reverse
---
All asm codes are compiled in [https://godbolt.org/](https://godbolt.org/), by `x86-64 ocamlopt 5.2.0`.

# let declaration

Let's start from a simple example. A whole file is written with only one line:

```ml Example.ml
let a = 10

let b = [1;2;3]
```

```asm
camlExample.data_begin:
camlExample.code_begin:
camlExample:
        .quad   1
        .quad   1
camlExample.gc_roots:
        .quad   camlExample
        .quad   0
camlExample.3:
        .quad   3
        .quad   camlExample.2
camlExample.2:
        .quad   5
        .quad   camlExample.1
camlExample.1:
        .quad   7
        .quad   1
camlExample.entry:
        movl    $21, %esi
        movq    camlExample@GOTPCREL(%rip), %rdi
        movq    %rsp, %rbx
        movq    64(%r14), %rsp
        call    caml_initialize@PLT
        movq    %rbx, %rsp
        movq    camlExample.3@GOTPCREL(%rip), %rsi
        movq    camlExample@GOTPCREL(%rip), %rdi
        addq    $8, %rdi
        movq    %rsp, %rbx
        movq    64(%r14), %rsp
        call    caml_initialize@PLT
        movq    %rbx, %rsp
        movl    $1, %eax
        ret
camlExample.code_end:
camlExample.data_end:
        .quad   0
camlExample.frametable:
        .quad   0
```

The value of the list is stored in segment.

The immediate value `$21` is the value of a `10`, I guess that may bring some benefit for arithmetic (And `21 >> 1 == 10`).

The `3` in `camlExample.3` is the `1`, and the `5` in `camlExample.2` is the `2`.
I can see the linked list structure that there's an address right after the integer data(and there's a `1` as zero standing for the tail).

After get the address of the variables, it calls the function `caml_initialize`, the 1st is `caml_initialize(camlExample, 10)` and the 2nd is `caml_initialize(camlExample+8, camlExample.3)`.
So I know the data in `camlExample` stores the pointer of the variables.
The official doc also [mentioned](https://ocaml.org/manual/5.3/intfc.html#ss:c-low-level-gc-harmony) it.

I also see that it `mov` a strange value `%r14 + 64` to `%rsp` to take place of previous `%rsp`. This is not easy to explain. In short, OCaml compiler use Emulating Thread Local Storage tech in gcc to let the OCaml processes run as threads of C process, which causes the variable declaration `caml_initialize` should use the main thread process stack address `%r14 + 0x40`. It doesn't matter and I can just ignore it(although I takes hours to make it up).

# curry

## Source code

```ml Example.ml
let sum_ints = 
    List.fold_left (fun acc x -> acc + x) 0

let res = sum_ints [1;2;3]
```

## Assembly

```asm
camlExample.data_begin:
camlExample.code_begin:
camlExample.4:
        .quad   caml_curry2
        .quad   0x200000000000007
        .quad   camlExample.fun_342
camlExample:
        .quad   1
        .quad   1
camlExample.gc_roots:
        .quad   camlExample
        .quad   0
camlExample.fun_342:
        leaq    -1(%rax,%rbx), %rax
        ret
camlExample.fun_348:
        movq    %rax, %rdi
        movq    24(%rbx), %rsi
        movq    16(%rbx), %rax
        movq    %rsi, %rbx
        jmp     camlStdlib__List.fold_left_380@PLT
camlExample.3:
        .quad   3
        .quad   camlExample.2
camlExample.2:
        .quad   5
        .quad   camlExample.1
camlExample.1:
        .quad   7
        .quad   1
camlExample.entry:
        leaq    -320(%rsp), %r10
        cmpq    40(%r14), %r10
        jb      .L103                 ; goto .L103 when space is not enough
.L104:
        movl    $1, %eax
        movq    camlExample.4@GOTPCREL(%rip), %rbx
        movq    camlStdlib__List@GOTPCREL(%rip), %rdi
        movq    200(%rdi), %rdi
        subq    $48, %r15
        call    caml_allocN@PLT
.L105:
        leaq    8(%r15), %rsi         ; heap + 8
        movq    $5367, -8(%rsi)       ; strange metadata
        movq    camlExample.fun_348@GOTPCREL(%rip), %rdx
        movq    %rdx, (%rsi)
        movabsq $72057594037927941, %rdx
        movq    %rdx, 8(%rsi)         ; $72057594037927941
        movq    %rbx, 16(%rsi)        ; camlExample.4
        movq    %rax, 24(%rsi)        ; zero (%eax == 1, 1 >> 1 == 0)
        movq    %rdi, 32(%rsi)        ; List.fold_left
        movq    camlExample@GOTPCREL(%rip), %rdi
        movq    %rsp, %rbx
        movq    64(%r14), %rsp
        call    caml_initialize@PLT
        movq    %rbx, %rsp
        movq    camlExample@GOTPCREL(%rip), %rax
        movq    (%rax), %rax          ; pointer of sum_ints expression
        movq    camlExample.3@GOTPCREL(%rip), %rdi
        movq    24(%rax), %rbx        ; zero
        movq    16(%rax), %rax        ; camlExample.4
        call    camlStdlib__List.fold_left_380@PLT
.L106:
        movq    camlExample@GOTPCREL(%rip), %rdi
        addq    $8, %rdi              ; camlExample + 8, pointer of res
        movq    %rax, %rsi            ; value of res
        movq    %rsp, %rbx
        movq    64(%r14), %rsp
        call    caml_initialize@PLT   ; caml_initialize(camlExample+8, %rsi)
        movq    %rbx, %rsp
        movl    $1, %eax
        ret
.L103:
        push    $33
        call    caml_call_realloc_stack@PLT
        popq    %r10
        jmp     .L104
camlExample.code_end:
camlExample.data_end:
        .quad   0
camlExample.frametable:
        .quad   2
        .quad   .L106
        .word   9
        .word   0
        .long   (.L107 - .) + 0
        .quad   .L105
        .word   11
        .word   1
        .word   5
        .byte   1
        .byte   4
        .long   (.L108 - .) + 0
.L107:
        .long   (.L110 - .) + 1
        .long   1053016
        .long   (.L111 - .) + 0
        .long   2107600
.L108:
        .long   (.L110 - .) + 0
        .long   1053016
.L109:
        .ascii  "/app/example.ml\0"
.L111:
        .long   (.L109 - .) + 0
        .ascii  "Example.res\0"
.L110:
        .long   (.L109 - .) + 0
        .ascii  "Example.sum_ints\0"
```

Because the interger in OCaml is `OCamlnum == realnum * 2 + 1`, The `leaq   -1(%rax,%rbx), %rax` in `camlExample.fun_342` is doing a plus operation. It's my `(fun acc x -> acc + x)`.

At [here the official doc](https://ocaml.org/docs/compiler-backend#the-impact-of-polymorphic-comparison) said `%r15` is used as a heap pointer in my case:  
`OCaml on x86_64 architectures caches the location of the minor heap in the %r15 register since it's so frequently referenced in OCaml functions.`

It's amazing that there's two ways to curry a function. The first is the `camlExample.4`, use a built-in function to curry my lambda function. Another is to build a sequence for a expression, and evaluate it.

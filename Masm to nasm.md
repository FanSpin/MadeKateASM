# Masm to nasm 
## With a worked example

| Feature | MASM | NASM |
| :---- | :---- | :---- |
| Platform | Windows-only (ships with Visual Studio) | Cross-platform (Windows, Linux, macOS) |
| License | Proprietary | Open-source (2-clause BSD) |
| Case sensitivity | Case-insensitive | Case-sensitive for labels/variables |
| Memory references | mov ax, word PTR \[bx\] or mov ax, table\[bx\] | Requires square brackets: mov ax, word \[bx\] |
| Variable types | Remembers declared type (e.g. var dw 0) | Does not store types; must specify size: mov word \[var\], 2 |
| Procedures | PROC / ENDP keywords | Plain labels \+ ret |
| Sections | .data, .code, .data? | section .data, section .text, section .bss |
| Data directives | BYTE, WORD, DWORD, QWORD | db, dw, dd, dq |
| Uninitialized data | stack DB 64 DUP(?) | stack resb 64 |
| Repetition | count DUP (value) | TIMES count |
| Macros | Powerful, Turing-complete; MACRO/ENDM | Simpler; %macro/%endmacro, params as %1, %2… |
| Floating-point regs | ST(0), ST(1) | st0, st1 |
| Memory models | Supports 16-bit memory models (SMALL, MEDIUM, etc.) | No memory models; flat model only |
| ASSUME directive | Supported | Not supported |
| OFFSET keyword | Required for address-of | Not needed; bare label is the address |
| High-level constructs | INVOKE, .IF/.WHILE, STRUCT, PROTO | None built-in; must define macros manually |
| Object formats | COFF/OMF (Windows) | ELF, COFF, PE, binary, Mach-O, etc. |

Rule of thumb: use MASM when you're building Windows binaries inside Visual Studio and want tight toolchain integration;   
use NASM when you need portability, a strict/simple Intel-syntax assembler, or are targeting non-Windows platforms.

Here's the same Win32 "print a string" program in both dialects, side by side. I've added comments flagging each translation point.

## MASM (Windows, ml /c hello.asm)

 .386 <br>
.model flat, stdcall <br>
include  kernel32.inc<br>
includelib kernel32.lib<br>
<br>
.data  <br>
msg     DB  'Hello, World\!', 10, 0       ; ← DB (uppercase)  <br> 
len     EQU \$ \- msg \- 1                  ; ← EQU (same in both)  <br>
written DD  ?                            ; ← DD ? (MASM syntax for uninitialized) <br>
<br>
.code<br>
main PROC                                       ; ← PROC ... ENDP wrapper    <br>

<br>
; WriteFile(stdout, msg, len, \&written, 0\)    <br>

mov  eax, \-11                              ; STD\_OUTPUT\_HANDLE   <br>
call GetStdHandle 
mov  ebx, eax                              ; save handle <br>
<br>
push 0 <br>
push OFFSET written                        ; ← OFFSET keyword (MASM) <br>
push len <br>
push OFFSET msg                            ; ← OFFSET keyword (MASM) <br>
push ebx  <br>
call WriteFile     ; ExitProcess(0) <br>
push 0 <br>
call ExitProcess<br>
main ENDP                                       ; ← ENDP END <br>
main                                    ; ← END \+ entry label     |
| :---- |

## NASM (same program, nasm \-f win32 hello.asm && ld \-e main hello.obj \-lkernel32)

 global main<br>
extern GetStdHandle<br>
extern WriteFile <br>
extern ExitProcess <br>
<br>
section .data                                   ; ← "section .data" (not just .data) <br>
msg     db  'Hello, World\!', 10, 0         ; ← db (lowercase) <br>
len     equ \$ \- msg \- 1 <br>
written dd  0                              ; ← dd 0 (NASM has no "?") <br>
<br>
section .text                                   ; ← .text, not .code <br>
main:                                           ; ← bare label, no PROC/ENDP<br>
; WriteFile(stdout, msg, len, \&written, 0\) <br>
mov  eax, \-11 <br>
call GetStdHandle <br>
mov  ebx, eax <br>
<br>
push 0 <br>
push written                               ; ← no OFFSET; label IS the address <br>
push len <br>
push msg                                   ; ← no OFFSET <br>
push ebx <br>
call WriteFile <br>
<br>
; ExitProcess(0) <br>
push 0 <br>
call ExitProcess     |
| :---- |

## Cheat-sheet of what changed

| \# | MASM | NASM | Why |
| :---- | :---- | :---- | :---- |
| 1 | .data / .code | section .data / section .text | NASM always uses section |
| 2 | DB, DD | db, dd | NASM is case-sensitive |
| 3 | DD ? | dd 0 | NASM has no ? initializer |
| 4 | OFFSET msg | msg | In NASM a bare label already means its address |
| 5 | main PROC … main ENDP | main: | NASM has no procedure block |
| 6 | END main | global main | Different way to mark entry \+ export |
| 7 | INVOKE WriteFile, … | manual push \+ call | INVOKE is MASM-only sugar |
| 8 | mov eax, DWORD PTR \[var\] | mov eax, dword \[var\] | PTR → size keyword before \[ |

Tip for translating: work top-down — swap the section headers first, then the data directives, then strip OFFSET/PTR/PROC/ENDP, and finally replace any INVOKE with explicit push/call.

## *is there a specific "chip" or "architecture" masm matches?*

No single chip — MASM targets the x86 architecture (both 32-bit IA-32 and 64-bit x86-64/AMD64). That's the entire Intel/AMD PC CPU family, from the original 8086 through modern Core/Ryzen.  
You pick the minimum instruction set with a directive at the top of the file:

| Directive | Unlocks |
| :---- | :---- |
| .386 | 32-bit registers, EAX… |
| .486 | SSE (basic) |
| .586 | SSE2, conditional moves |
| .686 | 64-bit mode (with ml64.exe) |
| .XMM / .XMM2 | SSE / SSE2 instructions |

So the "architecture" MASM matches is x86 / x86-64 — the same ISA that NASM and GAS also target. The difference is purely in syntax dialect and toolchain, not in which chip you're writing for. MASM simply doesn't support ARM, RISC-V, or any non-x86 architecture.  

---
layout: post
title: m68k-MBC notes
---

Resources:

[PCBWAY Project site](https://www.pcbway.com/project/shareproject/68k_MBC__a_3_ICs_68008_homebrew_computer.html)
[Tindie](https://www.tindie.com/products/denjhang/68k-mbc-a-3-ics-68008-homebrew-computer/)
[68k SBC](https://elephantandchicken.co.uk/stuffandnonsense/?p=927)
[vasm site](http://sun.hasenbraten.de/vasm/release/vasm.html)


Assembler Hello world

```asm
; IOS equates
IOBASE      EQU     $FFFFC              ; Address base for the I/O ports
EXCWR_PORT  EQU     IOBASE+0            ; Address of the EXECUTE WRITE OPCODE write port
EXCRD_PORT  EQU     IOBASE+0            ; Address of the EXECUTE READ OPCODE read port
STOPC_PORT  EQU     IOBASE+1            ; Address of the STORE OPCODE write port
SER1RX_PORT EQU     IOBASE+1            ; Address of the SERIAL 1 RX read port 
SYSFLG_PORT EQU     IOBASE+2            ; Address of the SYSFLAGS read port
SER2RX_PORT EQU     IOBASE+3            ; Address of the SERIAL 2 RX read port
USRLED_OPC  EQU     $00                 ; USER LED opcode
SER1TX_OPC  EQU     $01                 ; SERIAL 1 TX opcode
SETIRQ_OPC  EQU     $02                 ; SETIRQ opcode
SELDISK_OPC EQU     $09                 ; SELDISK opcode
SELTRCK_OPC EQU     $0A                 ; SELTRACK opcode
SELSECT_OPC EQU     $0B                 ; SELSECT opcode
WRTSECT_OPC EQU     $0C                 ; WRITESECT opcode
SER2TX_OPC  EQU     $10                 ; SERIAL 2 TX opcode
ERRDSK_OPC  EQU     $85                 ; ERRDISK opcode
RDSECT_OPC  EQU     $86                 ; READSECT opcode
SDMOUNT_OPC EQU     $87                 ; SDMOUNT opcode
FLUSHBF_OPC EQU     $88                 ; FLUSHBUFF opcode

; Common ASCII codes
cr          EQU     $0d                 ; Carriage return
lf          EQU     $0a                 ; Line feed
eos         EQU     0                   ; End of string

        ; ---- vectors (even addresses) ----
        org     $000000
        dc.l    _stack_top          ; initial SSP
        dc.l    _start              ; initial PC

        ; (you can fill more vectors as needed, else default to _halt)
        rept    62
        dc.l    _halt
        endr

        ; ---- code ----
        org     $000100             ; keep code away from vectors

_start:
        lea     _bss_start,a0
        lea     _bss_end,a1
        moveq   #0,d0
.clr:   cmpa.l  a0,a1
        beq.s   STARTMAIN
        move.l  d0,(a0)+
        bra.s   .clr

STARTMAIN:
    ; NOTE: the SSP is already set at $1000 by the IOS load routine
    lea     MESSAGE,a1
    bsr     puts
;    bra.s   STARTMAIN
    stop    #$2700
   
    
; =========================================================================== ;
;
; Send a string to the serial 1, A1 = pointer to the string.
; NOTE: Only D0.B and A1 are used
;
; =========================================================================== ;
puts:
    move.b  (a1)+,d0                    ; D0.B = current char to print
    cmp.b   #eos,d0                     ; Is it an eos?
    beq     puts_end                    ; Yes, jump
    move.b  #SER1TX_OPC,(STOPC_PORT).l  ; Write SERIAL 1 TX opcode to IOS
    move.b  d0,(EXCWR_PORT).l           ; Print current char
    jmp     puts
    
puts_end:
    rts

MESSAGE     DC.B    'HELLO WORLD! ', cr, lf, eos


_halt:  stop    #$2700              ; halt CPU

        ; ---- memory symbols ----
_bss_start:
        ; reserve bss here if needed (ds.b …)
_bss_end:

        ; simple downward-growing stack near top of RAM
_stack_top  equ $00080000           ; e.g., 512 KB RAM top (adjust to your SBC)

            END     START
```

Assemble (use vscode Amiga assembly extension assembler):
```bash
.vscode/extensions/prb28.amiga-assembly-1.8.13/resources/bin/linux/vasmm68k_mot -m68008 -Fsrec -s19  -o dmon.hex dmon.s
sed -i 's/$/\r/' dmon2.hex 
tr '\n' ' ' < dmon2.hex > dmon22.hex
```

s-loader on m68k-mbc requires line end

the output: 

```text
S00700007365673089
S1230000000800000000010000000146000001460000014600000146000001460000014629
S1230020000001460000014600000146000001460000014600000146000001460000014684
S1230040000001460000014600000146000001460000014600000146000001460000014664
S1230060000001460000014600000146000001460000014600000146000001460000014644
S1230080000001460000014600000146000001460000014600000146000001460000014624
S12300A0000001460000014600000146000001460000014600000146000001460000014604
S12300C00000014600000146000001460000014600000146000001460000014600000146E4
S12300E00000014600000146000001460000014600000146000001460000014600000146C4
S123010041F8014A43F8014A7000B3C8670420C060F843F8013661044E72270010194A0012
S1230120671213FC0001000FFFFD13C0000FFFFC4EF8011C4E7548454C4C4F20574F524C4C
S10D01404421200D0A004E7227002E
S804000100FA
```

EOL is 0xd

```bash
$ ascii-xfr -s -l 100 -c 5 dmon23.hex > /dev/ttyUSB0 
```

ASCII upload of "dmon23.hex"
Line delay: 100 ms, character delay 5 ms
This boots to execution of "Hello world"

Step 1
The output from 
```vasmm68k_mot -m68008 -Fsrec -s19  -o dmon.hex dmon.s```
requires processing
\n needs to be replaced with \r (0xa -> 0xd)
sLoader requires 24 bit addressing so the last line with S9 record needs to be replaced with S8 record.
the code starts at $100 so the last line needs to be replaced with S804000100FA

Step 2
vasm can generate 24bit addresses, use flag -s28 to vasm
then only EOL needs to be modified in output and start address needs to be modified

Step 3
ORG start in source is not applied in the output, use -exec flag to vasm

```vasmm68k_mot -m68008 -Fsrec -s28 -exec=_start -o dmon.hex dmon.s```

This outputs hex that can be fed into sLoader directly

```ascii-xfr -s -l 20  dmon_2.hex > /dev/cu.usbserial-0001```

On monitoring terminal:
```cat /dev/cu.usbserial-0001```

```68k-MBC - A091020-R140221
IOS - I/O Subsystem - S310121-R231021

IOS: Full HW configuration detected
IOS: Found GPE Option
IOS: CP/M Autoexec is OFF
IOS: Loading boot program... done
IOS: 68008 CPU is running from now

sLoad - S-record Loader - S180221-R150521
1024KB - Full HW configuration
Waiting input stream...

S00700007365673089
S224000000000800000000010000000146000001460000014600000146000001460000014628
S224000020000001460000014600000146000001460000014600000146000001460000014683
S224000040000001460000014600000146000001460000014600000146000001460000014663
S224000060000001460000014600000146000001460000014600000146000001460000014643
S224000080000001460000014600000146000001460000014600000146000001460000014623
S2240000A0000001460000014600000146000001460000014600000146000001460000014603
S2240000C00000014600000146000001460000014600000146000001460000014600000146E3
S2240000E00000014600000146000001460000014600000146000001460000014600000146C3
S009000073656731303026
S22400010041F8014A43F8014A7000B3C8670420C060F843F8013661044E72270010194A0011
S224000120671213FC0001000FFFFD13C0000FFFFC4EF8011C4E7548454C4C4F20574F524C4B
S20E0001404421200D0A004E7227002D
S804000100FA
sLoad: Flushing input buffer... done
sLoad: Starting Address: 0x000100

HELLO WORLD!
```



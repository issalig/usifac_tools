# CPC Basic and Z80 adventures

This is the documentation of my journey into learning Z80 assembly. It is not a tutorial on Z80, just a quick start guide to mess around Z80 and CPC.

These are some of resources I used to on my way to write this.

	- For a good Z80 tutorial you can visit https://www.chibiakumas.com/z80/index.php
	- Firmware http://www.cantrell.org.uk/david/tech/cpc/cpc-firmware/
	- A nice page with clearly explained Z80 instructions http://www.z80.info/z80code.htm
	- Locomotive basic 1.1 disassembly http://www.cpctech.org.uk/docs/basic.asm
	- CPC6128 operating system ROM http://www.cpctech.org.uk/docs/os.asm
	- German book about firmware https://k1.spdns.de/Vintage/Schneider
	- Markdown cheatsheet https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet
	- and many others ...

I have used Linux but on Windows you can also do it.
I will use WinAPE http://www.winape.net/downloads.jsp running under linux with WINE (sudo apt install wine).
Also I will use iDSK tools https://github.com/cpcsdk/idsk but you can use similar tools.

For hexadecimal numbers I will use &,#,$,h indistinctively or any other symbol normally used for that.

## BASIC

Now let's start and go back to your good old BASIC days and type the following code on WinAPE. If you are new to CPC world, welcome aboard!

```basic
10 REM Hello
20 PRINT "Hello World!"
```

Create a disk on WinAPE (File->Drive A->New blank disk  and set **hello.dsk** as name and format it) if you have not done it or use and existing one (File->Drive A->Insert Disc Image) and save the program with 
```basic
SAVE "hello.bas"
```

This will create a file with AMSDOS header (http://www.cpcwiki.eu/index.php/AMSDOS_Header) with the bytecodes for the BASIC program (https://cpctech.cpcwiki.de/docs/bastech.html)

If you want to save it as ascii (and without AMSDOS header) use
```basic
SAVE "hello.txt",a
```
Now, on WinAPE you can remove the disc with File->Drive A->Remove disk

And go to the command line to use iDSK tool to list the contents of the disk.
```bash
iDSK hello.dsk -l
 
DSK : hello.dsk
HELLO   .BAS 0
HELLO   .TXT 0
------------------------------------
```

Now, we can take a look inside a .BAS file and will report the size of the code in ascii

```
iDSK hello.dsk -b hello.bas

DSK : hello.dsk
Amsdos file : hello.bas
Taille du fichier : 35
10 REM Hello
20 PRINT "Hello World!"
```

Or just
```
iDSK hello.dsk -a hello.txt
DSK : hello.dsk
Amsdos file : hello.txt
Taille du fichier : 1024
10 REM Hello
20 PRINT "Hello World!"

------------------------------------
```

Of course we can also open it with our favourite text editor.

If you load a txt file it will be interpreted as BASIC

```basic
LOAD "hello.txt"
LIST
10 REM Hello
20 PRINT "Hello World!"
```

ASCII is nice and human readable but let's dive into the .BAS file which is much more interesting (Reference on  https://cpctech.cpcwiki.de/docs/bastech.html)

Again, I will use iDSK to extract the bytes
```
iDSK hello.dsk -h hello.bas
```

This command will remove the first 128 bytes (AMSDOS headers) and we have the following bytes:

```
0c 00                          ; length of data 12 (&0c) bytes
0a 00                          ; line number 10 (&0a)
c5                             ; c5 is REM code
20 48 65 6c 6c 6f              ; string ' Hello' , 20 is blank char and so on
00                             ; end of line marker

15 00                          ; length of data 21 (&15) bytes
14 00                          ; line number 20 (&14)
bf                             ; bf is PRINT code
20 22 48 65 6c 6c 6f           ; string ' "Hello'
20 57 6f 72 6c 64 21 22        ; string ' World!"'
00                             ; end of line marker
00 00                          ; length of line 0 is end of basic program

```


## Assembly


And now that we know some things about BASIC, even we get down to the internal representation of a BASIC program, we are gonna play like the BIG boys, we are gonna play with **Z80 assembly**.

Simplifying, assembly is a list of operation codes (mnemonics) and parameters where these parameters can be numbers or registers.

### Registers

The important Z80 8-bit registers are A, B, C, D, E, F, H and L and can store any number from 0 to 255. A set of important 16-bit registers are AF BC DE HL, IX, IY and can store a number from 0 to 65535.
There are more registers but for now is enough.

- **A** is also called the "accumulator". It is the primary register for arithmetic operations and accessing memory.


- **HL** The general 16 bit register, it's used pretty much everywhere you use 16 bit registers. It's most common uses are for 16 bit arithmetic and storing the addresses of stuff (strings, pictures, labels, etc.). Note that HL usually holds the original address while DE holds the destination address.

### Instructions
Regarding instructions you will encounter LD, CP, JP, JR, RET, CALL on the following example


- **LD**: I would seay this the main instruction, it LOADS or stores to a variable. A register will be used for one-byte numbers and for two-byte numbers, any two-byte register is fine, but HL is usually the best choice. 
    - ld a, 5 ; sets value 5 to accumulator register


- **CP**: Comparison. It compares origin value with register A and sets Z flag to 1 if same values.
    - cp 5    ; compares current value and accumulator

- **RET**: Pops the stack and jumps to the address stored in the stack. With and argument is a conditional instruction (ret z)
    - ret z   ; goes back if Z==1

- **JP**: Jump Absolute. Goes to the specified  adress
    - jp 0    ; goes to 0

- **JR**: Jump Relative. Similar to JP, but faster because it takes 2 bytes instead of 3. It is used to jump to near places +-127 bytes and allows relocatable routines :)
    - jr 1    ; goes to next instruction, useless here

- **org**: Defines where the code starts. References to addresses will change according to this. As seen , JR will not be affected by this value.
    - org &1200
    
- **equ**: Sets a value to a label. Think of it as a constant.
    - five equ 5
    
- **defb**: Sets memory to specified values.
    - defb 20              ; it will store one byte
    - defb "defb example"  ; it will store a sequence of bytes

### Comments
Comments are placed after **;** and I **encourage** you to use them now that you are starting to learn and later when you get experience so you annotate all the dark magic you write. Failing to do so, will get you **exciting** moments trying to guess what you wrote.


For more information on Z80 instructions I find this link extremely clear http://www.z80.info/z80code.htm

### Hello world from the assembly side
The following code is borrowed from the great http://www.chibiakumas.com prints the "Hello World!" string. I annotated this in order to better understand what it does and I think it will also help you.

```asm
; Got it from www.chibiakumas.com/z80/helloworld.php#LessonH1
; and completed with explanations for noobs like me

org &1200                ; our code will start at &1200

PrintChar   equ &bb5a    ; TXT OUTPUT Output a character or controlcode to the Text VDU
                         ; Info on firmware calls www.cpcwiki.eu/imgs/7/73/S968se14.pdf

ld hl, Message           ; load address of string in HL
call PrintString         ; print it

; print CR+LF
Newline:
	ld a,13              ; CR
	call Printchar
	ld a,10	             ; LF
	jp Printchar

; loop until last char (255) and print char
PrintString:
	ld a,(hl)            ; load char index stored in HL into A
	cp 255               ; A-255  if equal Z flag will be set
	ret z                ; returns if Z flag is set
	inc hl	             ; hl=hl+1
	call PrintChar       ; call BB5A
	jr PrintString

; it is always safe to put defb after code (especially ret or jump), so this data will not be executed
Message: db 'Hello World!',255   ;string is ended with 255 as end of string

```

Once you have taken a look to the asm code, let's put it into WinAPE (Assembler->Show assembler), copypaste it and then assemble it (Ctrl+F9). If all was correct it should give 0 errors.

WinAPE shows the binary digits generated for this program

```asm
WinAPE Z80 Assembler V1.0.13

000001  0000                ; Got it from www.chibiakumas.com/z80/helloworld.php#LessonH1
000002  0000                ; and completed with explanations for noobs like me
000004  0000  (1200)        org &1200                ; our code will start at &1200
000006  1200  (BB5A)        PrintChar   equ &bb5a    ; TXT OUTPUT Output a character or controlcode to the Text VDU
000007  1200                                         ; Info on firmware calls www.cpcwiki.eu/imgs/7/73/S968se14.pdf
000009  1200  21 1A 12      ld hl, Message           ; load address of string in HL
000010  1203  CD 10 12      call PrintString         ; print it
000012  1206                ; print CR+LF
000013  1206                Newline
000014  1206  3E 0D         	ld a,13              ; CR
000015  1208  CD 5A BB      	call Printchar
000016  120B  3E 0A         	ld a,10	             ; LF
000017  120D  C3 5A BB      	jp Printchar
000019  1210                ; loop until last char (255) and print char
000020  1210                PrintString
000021  1210  7E            	ld a,(hl)            ; load char index stored in HL into A
000022  1211  FE FF         	cp 255               ; A-255  if equal Z flag will be set
000023  1213  C8            	ret z                ; returns if Z flag is set
000024  1214  23            	inc hl	             ; hl=hl+1
000025  1215  CD 5A BB      	call PrintChar       ; call BB5A
000026  1218  18 F6         	jr PrintString
000028  121A                ; it is always safe to put defb after code (especially ret or jump), so this data will not be executed
000029  121A                Message
000029  121A  48 65 6C 6C    db 'Hello World!',255   ;string is ended with 255 as end of string
        121E  6F 20 57 6F 
        1222  72 6C 64 21 
        1226  FF 
```

The generated code goes from &1200 to &1226, having a size of &27 (39 in decimal) bytes.
The first instruction at address &1200 is 21 1A 12 that corresponds to LD HL,nn instruction (code &21) and address 121A. You can check codes at http://map.grauw.nl/resources/z80instr.php

And to run it call starting address (&1200) from BASIC as we set org &1200 in the asm code.
```basic
call &1200
```
If everything was fine, now you are seeing "Hello world!" on the screen.

Now save it with (size of the program is 39 (&27) bytes as commented before)
save "hello.bin",b,&1200,&27

Eject disc if necessary and ask iDSK what is this file about (-h stands for hex dump, not help :) ).

iDSK can disassemble the program but he does not know that there is a string starting at &1200, so it will interprete these bytes as Z80 instructions :) Take into account that these defb regions sholud not be executed. In this example there is a jump just before the defb and this help us to keep things right.


```
iDSK hello.dsk -z hello.bin
DSK : hello.dsk
Amsdos file : hello.bin
Taille du fichier : 39
1200 21 1A 12       LD HL,121A
1203 CD 10 12       CALL 1210
1206 3E 0D          LD A,0D
1208 CD 5A BB       CALL BB5A    ; TXT_OUTPUT
120B 3E 0A          LD A,0A
120D C3 5A BB       JP BB5A
1210 7E             LD A,(HL)
1211 FE FF          CP FF
1213 C8             RET Z
1214 23             INC HL
1215 CD 5A BB       CALL BB5A    ; TXT_OUTPUT
1218 18 F6          JR 1210
121A 48             LD C,B       ; This is a string but iDSK does not know it
121B 65             LD H,L       ; and interpret it as Z80 instructions
121C 6C             LD L,H
121D 6C             LD L,H       ; That's why it is recommended to place the defb
121E 6F             LD L,A       ; after the code or in a place where it cannot be 
121F 20 57          JR NZ,1278   ; reached. Look at the previous instruction &1218
1221 6F             LD L,A       ; It is a jump to &1210!
1222 72             LD (HL),D
1223 6C             LD L,H
1224 64             LD H,H
1225 21 FF 1A       LD HL,1AFF
```

If you want an easier way of having the binary. You can get it from iDSK and paste it in https://onlinedisassembler.com

```
iDSK hello.dsk -h hello.bin
DSK : hello.dsk
Amsdos file : hello2.bin
#0000 21 1A 12 CD 10 12 3E 0D CD 5A BB 3E 0A C3 5A BB | !.....>..Z.>..Z.
#0010 7E FE FF C8 23 CD 5A BB 18 F6 48 65 6C 6C 6F 20 | ....#.Z...Hello.
#0020 57 6F 72 6C 64 21 FF 00 00 00 00 00 00 00 00 00 | World!..........
#0030 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 | ................
#0040 1A 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 | ................
```

### Mixing asm and BASIC

Now we know a little bit of assembly and that we have also seen how a BASIC code is stored in memory.

So the next question is, could it be possible to "write" BASIC code from asm?
Yes, it is.

For this example I will use code borrowed from USIFAC card and we will write our HELLO.BAS directly from asm.

BASIC program start at &170

```asm
data_size equ 39                            ; size of BASIC file, we know it is 39 bytes
addr equ &170                               ; BASIC files normally start at 170, let's write in this area
org	&1200                                   ; and store our program in &1200

                                            ; we will use 16-bit registers (hl, de, ... to deal with memory addresses that take 2 bytes &AABB)
	ld	hl, basic_code                      ; hl = address where the basic code is stored, i.e. org + something but the assembler does it for us
	ld 	de, addr                            ; de = &170
	ld	bc, data_size                       ; bc = the data size
	ldir                                    ; ldir copies a block from hl address to de address of length bc (i.e. memcpy)
    
basic_code:
defb &0c,&00,&0a,&00,&c5,&20,&48,&65,&6c,&6c,&6f,&00,&15,&00,&14,&00,&bf
defb &20,&22,&48,&65,&6c,&6c,&6f,&20,&57,&6f,&72,&6c,&64,&21,&22,&00,&00,&00    
    
```

Now compile it, execute it with CALL, type LIST command and our Hello World! code will appear. Of course you can also RUN it.

```
CALL &1200
Ready
list
10 REM Hello
20 PRINT "Hello World!"
Ready
RUN
Hello World!
Ready
```


And what about runnig the BASIC program from asm, well, this will be in the next episode.

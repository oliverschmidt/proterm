# ProTERM without interrupts

## Usage

* In the __Install Modem Port__ dialog, the entry __Apple Super Serial Card__ must be choosen.
* In the __Install Printer Port__ dialog, the entry __Apple Super Serial Card__ must __not__ be choosen (choose __Pascal 1.1.1 Printer Driver__ instead).
* If an unusable installation keeps ProTERM from starting up, the file `PT3.BIOS` must be deleted.

## Super Serial Card

The [Apple II Super Serial Card](https://en.wikipedia.org/wiki/Apple_II_serial_cards#Super_Serial_Card_(Apple_Computer)) (SSC) is built around the [6551 Asynchronous Communications Interface Adapter](https://en.wikipedia.org/wiki/MOS_Technology_6551) (ACIA) serial I/O chip.

## 6551

The 6551 can store only a single received byte.
Therefore it is of utmost importance for the Apple II program to retrieve that byte from the 6551 before the next byte is completely received - as this next byte just overwrites the previous byte.
Rather simple Apple II programs try to check the 6551 receive status often enough to notice when a byte is completely received and retrieve it then.
Sophisticated Apple II programs usually have the 6551 trigger an interrupt when a byte is completely received.
The interrupt handler retrieves the byte from the 6551 and stores it in a ring buffer for the main program to read it later.

## 6551 Interface Emulation

A _true_ 6551 emulation behaves of course like the 6551 in every aspect.
In contrast, a 6551 __interface__ emulation has more freedom.
A major difference is usually that the interface emulation doesn't only store a single byte but a lot of bytes.
Often this isn't a deliberate decission but rather an implcit attribute of the data channel used to feed the emulated 6551.
Think of TCP or USB which have significant receive buffers.

A 6551 interface emulation just never overwrites the previous byte with the next byte.
It keeps the next byte in some large receive buffer until the Apple II program has retrieved the previous byte.

For a simple Apple II program that periodically checks the 6551 receive status, this works perfectly fine:
Everytime the program checks the 6551 receive status, it sees a completely received byte.
The program then retrieves the byte and the emulation loads the byte from the receive buffer into the emulated 6551.
This way the programs both runs with maximum speed and controls the emulation speed.
Usually that results in the unmodified(!) program receiving data much faster than with a true 6551.

But for a sophisticated Apple II program that uses interrupts, things aren't that nice.
The emulation needs to decide when to trigger an interrupt without knowing how fast the program retrives bytes from its ring buffer.
There are basically two approaches:
* Limit the interrupt rate to the byte receive rate of the true 6551.
The downside is that the program cannot receive data faster than with a true 6551.
* Don't limit the interrupt rate at all.
The downside is that bursts of received data larger than the ring buffer of the program cause the program to loose some data.

## AppleWin

The SSC emulation of [AppleWin](https://en.wikipedia.org/wiki/AppleWin) can work both as:
* True 6551 emulation using a host PC COM port as data channel
* 6551 interface emulation using TCP as data channel

The AppleWin help file says: 

_TCP mode doesn't throttle the serial data-rate, so after reading the Status register (to clear the Rx interrupt) the Rx interrupt may get asserted immediately if there is more data in the TCP receive buffer, resulting in a missed interrupt (and therefore missed Rx data)._

So it's obvious that AppleWin follows the unlimited interrupt rate approach.

## ProTERM

[ProTERM](https://en.wikipedia.org/wiki/ProTERM) is a sophisticated program that uses interrupts. Its interrupt handler stores received data in a 512 byte ring buffer.

So running ProTERM in AppleWin with the SSC emulation in TCP mode means that bursts of received data larger than 512 bytes can't be successfully received.
This can easily be verified with this Python script:

```python
import socket, time

data = bytes(b'0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz')

sock = socket.socket()
sock.connect(('localhost', 1977))

sent = 0
for i in range(10):
    sent += sock.send(data)
    print(sent)

time.sleep(1)
sock.close()
```

The script sends a burst of 620 bytes to the AppleWin SSC TCP port.
As the screenshot shows, most of those bytes are lost:

![old](https://github.com/oliverschmidt/ProTERM/assets/2664009/77bcad15-e4e1-44f6-b940-4dc14a230f12)

## ProTERM without interrupts

As described above, using interrupts is a big benefit for a true 6551 but rather a problem for a 6551 interface emulation.
Therefore it is desirable to modify ProTERM to not use interrupts in the first place for scenarios with such an emulation.
The following patches make ProTERM 3.1 not use interupts anymore:

In the file `PT3`...
```
Offset: $430A
Old: $78 $A6 $D6 $A5 $D7
New: $20 $4E $08 $80 $01

Offset: $705A
Old: $A9 $1D
New: $80 $08

Offset: $720E
Old: $2C $7E $C0 $2C $14
New: $A6 $D6 $A5 $D7 $60
```

In the file `PT3.CODE0`...
```
Offset: $16F9
Old: $88 $F0 $4A $29 $08 $F0 $22
New: $8F $C9 $08 $D0 $22 $80 $00

Offset: $1703
Old: $2C $16
New: $80 $09

Offset: $171C
Old: $B2 $D0 $8E $71 $C0 $FA $68 $40 $9C $09 $C0 $DA $A6 $00 $9C $71 $C0 $25 $CE $92 $D6 $E6 $D6 $D0 $06
New: $C4 $D6 $D0 $D5 $A6 $D6 $A5 $D7 $60 $A4 $D6 $88 $A6 $d6 $A5 $D7 $E4 $D9 $D0 $04 $C5 $DA $F0 $C1 $60

Offset: $179C
Old: $09
New: $0B
```

With these patches applied, ProTERM doesn't have any problem with the same burst of 620 bytes, as the screenshot shows:

![new](https://github.com/oliverschmidt/ProTERM/assets/2664009/e73611c1-fe82-45fc-ab1c-bb210e30ff06)

### Source Code

With these patches applied, the relevant source code looks like this (`$D6/$D7` is the ringbuffer write pointer, `$D9/$DA` is the ringbuffer read pointer):

```
630A-   20 4E 08    JSR   $084E
```
```
081E-   AD 89 C0    LDA   $C0n9
0821-   29 8F       AND   #$8F
0823-   C9 08       CMP   #$08
0825-   D0 22       BNE   $0849
0827-   80 00       BRA   $0829
0829-   AD 88 C0    LDA   $C0n8
082C-   80 09       BRA   $0837
```
```
0837-   25 CE       AND   $CE
0839-   92 D6       STA   ($D6)
083B-   E6 D6       INC   $D6
083D-   D0 06       BNE   $0845
083F-   A5 D7       LDA   $D7
0841-   49 01       EOR   #$01
0843-   85 D7       STA   $D7
0845-   C4 D6       CPY   $D6
0847-   D0 D5       BNE   $081E
0849-   A6 D6       LDX   $D6
084B-   A5 D7       LDA   $D7
084D-   60          RTS
084E-   A4 D6       LDY   $D6
0850-   88          DEY
0851-   A6 D6       LDX   $D6
0853-   A5 D7       LDA   $D7
0855-   E4 D9       CPX   $D9
0857-   D0 04       BNE   $085D
0859-   C5 DA       CMP   $DA
085B-   F0 C1       BEQ   $081E
085D-   60          RTS
```

Notes:

* The 6551 is only checked for new data if the ringbuffer is completely empty.
* More than 255 bytes are never written to the ringbuffer (although it has a size of 512 bytes).

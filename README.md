# lsp-emu

Emulation for the Boss MB87837 (LSP) chip, used in the SC-88Pro, SC-8850, ME-6, ME-6b, ME-X, and others.

## Chip Info

The LSP is a custom DSP which can execute 384 instructions programs for each sample. It can work with multiple sample rates (32k or 44.1k) and multiple bit depths. It supports two stereo outputs and one stereo input. It has a small internal ram for instructions, parameters and delay lines, but it also supports a bigger external one, used for more complex effects.

![lsp](./lsp_chip.jpeg)

### Pinout

- 7, 33, 47, 73: +5v VDD
- 2, 12, 17, 23, 29, 39, 42, 52, 63, 69: VSS

- 40, 41: XTAL-EXTAL, clock crystal interface, freq = 2 * 384 * sample rate

- 75: SYI, sync interface (TODO)
- 74: SYO, sync interface (TODO)

- 34, 35, 36, 37, 38, 43, 44, 45: CD0-7, host interface data I/O
- 32, 31, 30, 28: CA0-3, host interface address in
- 24: RXW, host interface write control in
- 25: DS, host interface read control in
- 26: XCS, host interface chip select in
- 27: INT, host interface interrupt (TODO)

- 6, 5, 9, 8, 80: ED0-4, external RAM data I/O
- 22, 21, 20, 19, 18, 16, 15, 14, 13, 1: EA0-9, external RAM address out
- 10, 11: CAS0-1, external RAM column address strobe
- 3: RAS, external RAM row address strobe
- 4: WE, external RAM write enable

- 65: DABCLK, bit clock for audio out
- 68: DALRCLK, word clock for audio out
- 66, 67: TRS0-1, serial data lines for audio out

- 78: ADBCLK, bit clock for audio in
- 79: ADCC, word clock for audio in
- 77: TRR, serial data line for audio in
- 76: CK (TODO)

- 58, 59, 60, 61, 62, 64: G-XG2-XG1-A-B-C, address decoder control in
- 49, 50, 51, 53, 54, 55, 56, 57: Q0-7, address decoder out
- 46, 48: CPUCK1-2 (TODO)

- 70: X68 (TODO)
- 71: XE (TODO)
- 72: TEST (TODO)

### Host Interface

- **Read ready status**: read 0x03 until it reads 0x00
- **Write config register**: wait for ready status, write config in 0x03-0x02 then write 0x00 in 0x06
- **Write internal memory**: wait for ready status, write 3 bytes in 0x04(MSB)-0x03-0x02(LSB), then address in 0x01-0x00, writing in 0x00 triggers the write
- **Read internal memory**: write address in 0x09-0x08, wait for ready status, read 3 bytes from 0x02(MSB)-0x01-0x00(LSB)

### Config register

- 0x0001: start DSP
- 0x1001: halt DSP (18 bit? ME-6)
- 0x1021: halt DSP (16 bit? SC-88Pro)

### Internal Memory

All internal variables are 24 bit.

- 0x000-0x07f: Internal delay line/registers
  - The internal registers are typically accessed as a circular buffer. Every sample the current memory pointer is decremented by 1, and most(all?) instructions access memory access it using an offset
- 0x080-0x1ff: DSP program (24 bit x 384 instr)
  - Cannot(?) be written by the DSP program


## DSP instructions format

`ii rr cc`

```
ii[7:4] (0xf0):  opcode
ii[3]   (0x08):  mem[offset] <- acc
ii[2]   (0x04):  external ram?
ii[1]   (0x02):  external ram?
ii[0]   (0x01):  external ram?

rr[7]:         scale select
rr[6:0]:       mem offset/shifter

cc[7:0]:       immediate/coefficient (int8_t)
```

### Pipeline

The chip is pipelined, so the order/location of instructions is important. Programs have NOPs (`00 00 00`) to flush the pipeline pretty regularly. Some programs also use the order instructions to perform side effects (for example reassigning a variable and reading the old value immediately after).

- The write instructions (like `08 00 00`) will use the accumulator value skipping two slots before
- An increment immediate instruction (like `00 01 01`) can be placed immediately after a load, MAC or load results seems to be applied immediately
- Written memory can be accessed immediately after writing
- External ram needs NOPs (TODO)
- Instructions with special registers (`c0 rr xx`) have special needs
  - `c0 40 xx` will read the accumulator 3 instructions before (like `08`)
  - `c0 50 xx` and `c0 d0 xx` will read the accumulator that is set in the instruction immediately before
  - `c0 50 xx` will set the accumulator after the following instruction has completed


### Instr 00/08/20/28/c0/c8: MAC

`ab rr cc`

```
a(in_a): 00: in_a=acc      20: in_a=0        c0: in_a=0
b:       00: nothing       08: store to mem
rr[7]:   scale select
rr[6:0]: mem offset/shifter
cc:      immediate/mul coefficient (int8_t)
```

Increments (opcode 0x) or replaces (opcode 2x,Cx) the accumulator.
The shifter values of 1/2/3/4 work as immediates, every other works as a memory reference offset. Increment works with 24 bit saturation and multiplies using fractional integer multiplication. The scale of the multiplication can be set with rr[7].
Can also store the current value to the ram before computing the new one (opcode x8).
Opcode Cx works similarly to 2x, but doesn't replace the accumulator when using shifter values of 1/2/3/4. It also allows reading/writing the audio input/output at address 7f, whereas the other opcode don't work for that. It also can use the accumulator

Examples:
  - `00 00 7f`: acc += $00 * (0x7f/0x80)
  - `00 01 7f`: acc += 0x7f << 0
  - `00 02 7f`: acc += 0x7f << 5
  - `00 03 7f`: acc += 0x7f << 10
  - `00 04 7f`: acc += 0x7f << 15
  - `00 05 7f`: acc += $05 * (0x7f/0x80)
  
  - `00 80 7f`: acc += $00 * (0x7f/0x20)
  - `00 81 7f`: acc += 0x7f << 2
  - `00 82 7f`: acc += 0x7f << 7
  - `00 83 7f`: acc += 0x7f << 12
  - `00 84 7f`: acc += 0x7f << 17   (warning: saturated to 0x7fffff)
  - `00 85 7f`: acc += $05 * (0x7f/0x20)

  - `00 00 90`: acc += $00 * (signed(0x90)/0x80)
  - `00 01 90`: acc += signed(0x90) << 0
  - `00 02 90`: acc += signed(0x90) << 5
  - `00 03 90`: acc += signed(0x90) << 10
  - `00 04 90`: acc += signed(0x90) << 15
  - `00 05 90`: acc += $05 * (signed(0x90)/0x80)

  - `20 00 7f`: acc = $00 * (0x7f/0x80)
  - `20 01 7f`: acc = 0x7f << 0
  - `20 02 7f`: acc = 0x7f << 5
  - `20 03 7f`: acc = 0x7f << 10
  - `20 04 7f`: acc = 0x7f << 15
  - `20 05 7f`: acc = $05 * (0x7f/0x80)

  - `c0 00 7f`: acc = $00 * (0x7f/0x80)
  - `c0 01 7f`: acc += 0x7f << 0
  - `c0 02 7f`: acc += 0x7f << 5
  - `c0 03 7f`: acc += 0x7f << 10
  - `c0 04 7f`: acc += 0x7f << 15
  - `c0 05 7f`: acc = $05 * (0x7f/0x80)

C0 special cases:
  - `c0 40 7f`: acc = acc + acc * (0x7f/0x80)
  - `c0 50 ff`: acc = acc + (acc >> 5) * (0xff/0x80)
  - `c0 d0 ff`: acc = acc + (acc >> 5) * (0xff/0x20)
  - `c0 7f 7f`: acc = audio_in * (0x7f/0x80)

### Audio I/O

`c0 7f 7f` -> input audio into acc
addr 7e/7f work the same?
ff works the same but changes multiplier scale

input is dependent on pgm pos
- `>2   -> in channel 2`
- `>191 -> in channel 1`


`c8 58 00` -> output acc into audio out
dependent on pgm pos
- `>2   -> out channel 2` (TODO check)
- `>191 -> out channel 1` (TODO check)

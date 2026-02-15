# AT90USB1286 Core Notes for simavr

## Key Differences from ATmega32U4

| Feature | ATmega32U4 | AT90USB1286 |
|---------|-----------|-------------|
| Flash | 32KB | 128KB |
| SRAM | 2.5KB | 8KB |
| EEPROM | 1KB | 4KB |
| Ports | B,C,D,E,F | A,B,C,D,E,F |
| Timers | 0,1,3 (+ 4 high-speed) | 0,1,2,3 |
| Timer1 COMPC | No | Yes (OC1C=PB7) |
| Timer3 COMPC | No | Yes (OC3C=PC4) |
| Timer2 (8-bit async) | No | Yes (with ASSR/AS2) |
| External Interrupts | INT0-3, INT6 | INT0-7 |
| USART | USART1 | USART1 |
| USB Endpoints | 5 (0-4) | 7 (0-6) |
| ADC MUX bits | 6 (MUX0-4 + MUX5) | 5 (MUX0-4 only) |
| SPM_PAGESIZE | 128 | 256 |
| Vector size | 4 bytes | 4 bytes |
| Signature | 0x1E 0x95 0x87 | 0x1E 0x97 0x82 |

## Memory Map (from iousb1286.h)

- RAMSTART = 0x100
- RAMEND = 0x20FF (8KB SRAM)
- FLASHEND = 0x1FFFF (128KB)
- E2END = 0xFFF (4KB EEPROM)
- SPM_PAGESIZE = 256

## I/O Ports

- **Port A** (PA0-PA7): General I/O, AD0/PA0 also used as ADC input
- **Port B** (PB0-PB7): PCINT0-7, SPI (PB0=SS, PB1=SCK, PB2=MOSI, PB3=MISO)
  - PB4 = OC2A
  - PB5 = OC1A
  - PB6 = OC1B
  - PB7 = OC0A / OC1C (shared)
- **Port C** (PC0-PC7):
  - PC3 = T3 (Timer3 external clock)
  - PC4 = OC3C
  - PC5 = OC3B
  - PC6 = OC3A
  - PC7 = ICP3 / CLKO
- **Port D** (PD0-PD7):
  - PD0 = INT0 / SCL / OC0B
  - PD1 = INT1 / SDA / OC2B
  - PD2 = INT2 / RXD1
  - PD3 = INT3 / TXD1
  - PD4 = ICP1
  - PD6 = T1 (Timer1 external clock)
  - PD7 = T0 (Timer0 external clock)
- **Port E** (PE0-PE7):
  - PE4 = INT4 / TOSC1
  - PE5 = INT5 / TOSC2
  - PE6 = INT6 / AIN0
  - PE7 = INT7 / AIN1
- **Port F** (PF0-PF7): ADC0-ADC7

## External Interrupts

| INT | Pin | Control Register |
|-----|-----|-----------------|
| INT0 | PD0 | EICRA (A) |
| INT1 | PD1 | EICRA (A) |
| INT2 | PD2 | EICRA (A) |
| INT3 | PD3 | EICRA (A) |
| INT4 | PE4 | EICRB (B) |
| INT5 | PE5 | EICRB (B) |
| INT6 | PE6 | EICRB (B) |
| INT7 | PE7 | EICRB (B) |

EICRA at 0x69 (ISC00-ISC31), EICRB at 0x6A (ISC40-ISC71).

## Pin Change Interrupts

Only one bank: PCINT0-7 on Port B (PB0-PB7).
- PCICR bit: PCIE0
- PCIFR bit: PCIF0
- Vector: PCINT0_vect
- Mask register: PCMSK0

## Timers

### Timer0 (8-bit)
- PRR0 bit: PRTIM0
- OC0A = PB7, OC0B = PD0
- T0 (ext clock) = PD7
- cs_div: standard {0, 0, 3, 6, 8, 10, ext_fall, ext_rise}
- WGM modes: Normal(0), CTC(2), FastPWM(3), OCPWM(7)

### Timer1 (16-bit, 3 compare channels)
- PRR0 bit: PRTIM1
- OC1A = PB5, OC1B = PB6, OC1C = PB7 (shared with OC0A)
- ICP1 = PD4, T1 (ext clock) = PD6
- Has FOC1A, FOC1B, FOC1C in TCCR1C
- cs_div: standard {0, 0, 3, 6, 8, 10, ext_fall, ext_rise}

### Timer2 (8-bit async)
- PRR0 bit: PRTIM2
- OC2A = PB4, OC2B = PD1
- Async source: AS2 bit in ASSR register
- cs_div for async: {0, 0, 3, 5, 6, 7, 8, 10} (dividers: 1, 8, 32, 64, 128, 256, 1024)
- WGM modes: Normal(0), CTC(2), FastPWM(3), OCPWM(7)
- No external clock pin (TOSC1/TOSC2 for 32kHz crystal on PE4/PE5)

### Timer3 (16-bit, 3 compare channels)
- PRR1 bit: PRTIM3
- OC3A = PC6, OC3B = PC5, OC3C = PC4
- ICP3 = PC7, T3 (ext clock) = PC3
- cs_div: standard {0, 0, 3, 6, 8, 10, ext_fall, ext_rise}

## ADC

### ADMUX Register
- REFS1:0 (bits 7:6): Reference selection
  - 00: AREF, Internal Vref turned off
  - 01: AVCC with external capacitor on AREF pin
  - 10: Reserved
  - 11: Internal 2.56V with external capacitor on AREF pin
- ADLAR (bit 5): Left adjust result
- MUX4:0 (bits 4:0): Channel/gain selection (NO MUX5 -- only 5 mux bits, 32 entries)

### ADCSRB Register
- ADHSM (bit 7): ADC High Speed Mode
- ACME (bit 6): Analog Comparator Multiplexer Enable
- ADTS2:0 (bits 2:0): ADC Auto Trigger Source (only 3 bits, no ADTS3)

### ADC Mux Table (Table 25-4)

| MUX4:0 | Single | Pos Diff | Neg Diff | Gain |
|--------|--------|----------|----------|------|
| 00000 | ADC0 | - | - | - |
| 00001 | ADC1 | - | - | - |
| 00010 | ADC2 | - | - | - |
| 00011 | ADC3 | - | - | - |
| 00100 | ADC4 | - | - | - |
| 00101 | ADC5 | - | - | - |
| 00110 | ADC6 | - | - | - |
| 00111 | ADC7 | - | - | - |
| 01000 | - | ADC0 | ADC0 | 10x |
| 01001 | - | ADC1 | ADC0 | 10x |
| 01010 | - | ADC0 | ADC0 | 200x |
| 01011 | - | ADC1 | ADC0 | 200x |
| 01100 | - | ADC2 | ADC2 | 10x (reserved) |
| 01101 | - | ADC3 | ADC2 | 10x |
| 01110 | - | ADC2 | ADC2 | 200x |
| 01111 | - | ADC3 | ADC2 | 200x |
| 10000 | - | ADC0 | ADC1 | 1x |
| 10001 | - | ADC1 | ADC1 | 1x (self-ref) |
| 10010 | - | ADC2 | ADC1 | 1x |
| 10011 | - | ADC3 | ADC1 | 1x |
| 10100 | - | ADC4 | ADC1 | 1x |
| 10101 | - | ADC5 | ADC1 | 1x |
| 10110 | - | ADC6 | ADC1 | 1x |
| 10111 | - | ADC7 | ADC1 | 1x |
| 11000 | - | ADC0 | ADC2 | 1x |
| 11001 | - | ADC1 | ADC2 | 1x |
| 11010 | - | ADC2 | ADC2 | 1x (self-ref) |
| 11011 | - | ADC3 | ADC2 | 1x |
| 11100 | - | ADC4 | ADC2 | 1x |
| 11101 | - | ADC5 | ADC2 | 1x |
| 11110 | 1.1V bandgap | - | - | - |
| 11111 | 0V (GND) | - | - | - |

### ADC Auto Trigger Sources (Table 25-6)

| ADTS2:0 | Trigger Source |
|---------|---------------|
| 000 | Free Running mode |
| 001 | Analog Comparator |
| 010 | External Interrupt Request 0 |
| 011 | Timer/Counter0 Compare Match A |
| 100 | Timer/Counter0 Overflow |
| 101 | Timer/Counter1 Compare Match B |
| 110 | Timer/Counter1 Overflow |
| 111 | Timer/Counter1 Capture Event |

### Analog Comparator Mux

When ACME=1 and ADEN=0, MUX2:0 in ADMUX selects the negative input:
- 000=ADC0, 001=ADC1, 010=ADC2, 011=ADC3, 100=ADC4, 101=ADC5, 110=ADC6, 111=ADC7
- 8 mux inputs total

## USB

- USBCON at 0xD8, UDCON at 0xE0
- 7 endpoints (0-6), endpoint reset register UERST has EPRST0-6
- USB_COM_vect and USB_GEN_vect interrupt vectors
- PRR1 bit: PRUSB
- PLLCSR for PLL control
- USBRF (bit 5 of MCUSR) -- not defined in header, must be manually defined

## Interrupt Vectors (from iousb1286.h)

38 vectors total, 4 bytes each (152 bytes of vector table).

## Power Reduction Registers

- PRR0: PRADC, PRSPI, PRTIM1, PRTIM0, PRTIM2, PRTWI (+ reserved bits)
- PRR1: PRUSB, PRTIM3, PRUSART1

## RAMPZ

Extended program memory access register at address defined by RAMPZ constant.
Required for >64KB flash addressing (LPM/ELPM instructions).

## Missing Header Definitions

- `USBRF` (bit 5 of MCUSR) is NOT defined in `avr/iousb1286.h` -- must add `#define USBRF 5` before including the header

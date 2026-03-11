---
title: "Project TELEGRAPH (Part 3): Bare Metal, Bit-Banging, and the 4MB Heist"
author: Kamil Wcisło
date: 2026-03-11T12:00:00+01:00
draft: true
toc: true
series: telegraph
tags: ["Telegraph", "Electronics", "Reverse engineering", "arm"]
thumbnail: images/20260311/small_screen_close.webp
summary: |
    With our custom ESP32 wireless flasher built and tested in Part 2, we finally severed the USB cord.
    Now it was time to actually run some code on the clock's STM32F105RB.
---

I expected to just load up an Arduino "Blink" sketch and start mapping pins. The toolchain had other plans.

## 1. Bootstrapping the Bare Metal (The PlatformIO Struggle)

The STM32F105 "Connectivity Line" is the weird middle child of the ST lineup, less powerful than the F107-line, more
like F103, but with full USB controller (with host mode). It was used for downloading data from the drive - neat,
could be useful. When I tried to compile a basic PlatformIO project, I hit a wall: the F105RB is actually mocked in
standard PlatformIO Arduino definitions. It lacks linker script, proper board definition and some other things. This
is probably due to the fact, that there is no popular development board with F105 that I know...

To get code to compile, I created a custom board manifest
([genericSTM32F105RB.json](https://github.com/mek-x/hazk-fw/blob/main/boards/genericSTM32F105RB.json)) and copied the
[linker script](https://github.com/mek-x/hazk-fw/blob/main/boards/ldscript.ld) from some other board with F103 chip -
those should be almost the same due to similarities between memory layouts for those chip families.

But compiling is only half the battle. When I flashed the firmware, I got the "Silent Brick." The MCU did absolutely
nothing. No UART output, no GPIO toggling.

The culprit? The clock. It seems the initialization for the F105 is [totally
missing](https://github.com/stm32duino/Arduino_Core_STM32/blob/6b2b23d7e611d9025eb55145b45e0b5f5fbf5ee6/variants/STM32F1xx/F105R(8-B-C)T/generic_clock.c).

To fix this, I had to bypass the framework's initialization and write a custom `SystemClock_Config`. I manually
forced the microcontroller to use its High-Speed Internal (HSI) oscillator and configured the Phase-Locked Loop (PLL) to
hit our target frequency. This is the snippet of the clock init:

```c
// Bypassing the Arduino core to force the internal oscillator
extern "C" void SystemClock_Config(void) {
  RCC_OscInitTypeDef RCC_OscInitStruct = {0};
  RCC_ClkInitTypeDef RCC_ClkInitStruct = {0};

  // 1. Setup HSI and Main PLL
  RCC_OscInitStruct.OscillatorType = RCC_OSCILLATORTYPE_HSI;
  RCC_OscInitStruct.HSIState = RCC_HSI_ON;
  RCC_OscInitStruct.HSICalibrationValue = RCC_HSICALIBRATION_DEFAULT;

  // On F105, HSI is always divided by 2 before reaching the PLL
  // 8MHz / 2 = 4MHz.
  // 4MHz * 9 = 36MHz (Let's start safe at 36MHz)
  RCC_OscInitStruct.PLL.PLLState = RCC_PLL_ON;
  RCC_OscInitStruct.PLL.PLLSource = RCC_PLLSOURCE_HSI_DIV2;
  RCC_OscInitStruct.PLL.PLLMUL = RCC_PLL_MUL9;

  if (HAL_RCC_OscConfig(&RCC_OscInitStruct) != HAL_OK) {
    while(1); // Hang here if config fails
  }

  // 2. Configure Clock Tree
  RCC_ClkInitStruct.ClockType = RCC_CLOCKTYPE_HCLK|RCC_CLOCKTYPE_SYSCLK
                              |RCC_CLOCKTYPE_PCLK1|RCC_CLOCKTYPE_PCLK2;
  RCC_ClkInitStruct.SYSCLKSource = RCC_SYSCLKSOURCE_PLLCLK;
  RCC_ClkInitStruct.AHBCLKDivider = RCC_SYSCLK_DIV1;
  RCC_ClkInitStruct.APB1CLKDivider = RCC_HCLK_DIV2; // APB1 must be <= 36MHz
  RCC_ClkInitStruct.APB2CLKDivider = RCC_HCLK_DIV1;

  if (HAL_RCC_ClockConfig(&RCC_ClkInitStruct, FLASH_LATENCY_1) != HAL_OK) {
    while(1);
  }
}
```

> It's from [this file](https://github.com/mek-x/hazk-fw/blob/main/src/sysclk.cpp) in the project.

Suddenly, the chip woke up.

## 2. "Ping..." (Bridging the Gap)

With the clock tree stable, I routed the UART output through the ESP32 wireless serial bridge (`ser2net`) we built in
the last post.

When I've opened a TCP socket to the ESP32 on my laptop
[`Ping...`](https://github.com/mek-x/hazk-fw/blob/fae237aaa9fb35f2b51408f9fd085356020ffde6/src/main.cpp) was popping up
on my terminal. The chip's heart was beating. We had full, untethered code execution and debugging across the room.

Additionally I was changing the state on `PC6` pin, since it was one of those I had available on some connector and I
could check that it's state was changing using the multimeter (if UART would've not worked).

## 3. Cracking the Matrix (The Display Drivers)

Now for the fun part: making it glow. Board consists of basically 3 displays:

- the 7-segment (12 digits) LED display driven by **TM1629A** chip,
- 21x14 green led matrix driven by 3 **SM16126D** shift registers,
- 70x14 red led matrix driven by 6 **SM16126D** shift registers,

Connections to the matrices are well described:

![Screen signal ribbon connector](images/20260311/connections.webp)

### The TM1629A (The 7-Segment Driver)

![TM1629A](images/20260311/tm1629.webp)

Tracing the traces from the 7-segment displays led me to a
[**TM1629A**](https://bafnadevices.com/wp-content/uploads/2024/04/TM1629A_V2.1_EN.pdf). This is a dedicated LED drive
control IC that handles its own internal multiplexing. It doesn't need constant babying from the CPU; you just send it a
serial command containing the digits you want to display, and it latches them onto the 7-segment screen. First I had to
check for the 3 signals used by this chip (_strobe_, _clock_ and _data in/out_). I've used multimeter in continuity test
mode and I've identified fo9llowing connections leading to my MCU:

| Function | TM's Pin | MCU |
| -------- | -------- | --- |
| DIO      | 7        | PB4 |
| CLK      | 8        | PB3 |
| STB      | 9        | PB5 |

Next I had to identify the addressing. It was interesting, since **TM1629A** supports 16 digits with 8 segments (7
segments + dot), the connection wasn't straightforward. I've written [this
scanner](https://github.com/mek-x/hazk-fw/blob/b86d7d5bb9ac4f1ddc60958cbce8287f3586bd80/src/main.cpp) to identify proper
addressing. It led me to implement [this driver](https://github.com/mek-x/hazk-fw/blob/main/src/tm1629a.cpp).

The most interesting part is this function:

```c
// Converts digitIdx to actual address used by TM1629
// NOTE: 10, 11 are not used (there is hole in addressing)
const uint8_t digitToBit[] = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 12, 13};

/* for segData, bit means which segment from A-G is lit, e.g:
               pGFEDCBA
   0 → 0x3F (0b00111111)
   1 → 0x06 (0b00000110)
   2 → 0x5B (0b01011011)
   3→  0x4F (0b01001111)

      -A-
     |   |
     F   B
     |   |
     +-G-+
     |   |
     E   C
     |   |
      -B-  p

NOTE: dp (decimal point) is not used, we don't have any decimal points on our display
*/
void tm_setDigitRaw(uint8_t digitIdx, uint8_t segData) {
  if (digitIdx > 11) return;

  uint8_t bitIdx = digitToBit[digitIdx];
  uint8_t byteOffset = (bitIdx >= 8) ? 1 : 0;
  uint8_t bitMask = 1 << (bitIdx % 8);

  // Loop through segments A(0) to G(6)
  for (uint8_t seg = 0; seg < 7; seg++) {
    // TM1629A Addresses: A=0x00, B=0x02, C=0x04, D=0x06, E=0x08, F=0x0A, G=0x0C
    uint8_t addr = (seg * 2) + byteOffset;

    if (segData & (1 << seg)) {
      tm_framebuffer[addr] |= bitMask;  // Turn ON segment
    } else {
      tm_framebuffer[addr] &= ~bitMask; // Turn OFF segment
    }
  }

  // `tm_framebuffer` is written to chip's registers during `tm_updateDisplay` call
}
```

### The SM16126D (The LED Matrix)

![SM16126D](images/20260311/sm16126.webp)

The main dot-matrix display was a completely different beast. Tracing these lines led to a bank of **SM16126D** ICs.
These are effectively high-power shift registers. Unlike the **TM1629A**, these chips have no memory and no internal
multiplexer. It seems, that first of the **SM16126D**'s was controlling rows and rest of them were responsible for their
set of columns.

Using multimeter, I've identified the connections:

| P3 Pin           | MCU  | First SM16126D Pin |
| ---------------- | ---- | ------------------ |
| CLK              | PB12 | 3                  |
| OE               | PB13 | 21                 |
| STB              | PB14 | 4                  |
| DIN              | PA13 | 2                  |
| DIN (big screen) | PB15 | 2                  |

To light up the matrix, I had to do the heavy lifting in software. I wrote a [custom C++ multiplexing
loop](https://github.com/mek-x/hazk-fw/blob/503dcd67c79e17c547e808becc58a79603a0a2a9/src/main.cpp) to push raw
framebuffers into the shift registers. Because the **SM16126D**'s forget their state the moment you move to the next
row, this code had to be fast.

We now had complete programmatic control over both the 7-segment readout and the raw LED matrix, all driven from our
custom bare-metal firmware.

## 4. The Mystery 5-Pin Device (The Hardware Trap)

While tracing the board, I found an unknown 5-pin footprint. The pins mapped to PC6, PC7, a 3.3V line, Ground, and a trace leading directly to a CR2032 coin cell battery.

The battery pin was the smoking gun: this was a Real-Time Clock (RTC).

I assumed it was I2C. However, here is the classic STM32 hardware trap: on the F105, PC6 and PC7 do not have hardware I2C support mapped to them. The original engineers just picked two random GPIOs to save board routing space.

Since I couldn't use the hardware Wire library (it would freeze looking for non-existent peripherals), I wrote a custom, auto-detecting "Bit-Bang" I2C scanner. It manually toggled the GPIOs HIGH and LOW to simulate the open-drain I2C protocol. It swept the bus and immediately got an ACK on address 0x68.

It was a Maxim DS3231 RTC and temperature sensor. A quick read confirmed it: it spit out the factory epoch time (2000-01-01 04:48:51) and the live ambient temperature of my office (23.00 C).

## 5. Engaging the Final Boss (The Winbond SPI Flash)

The last major component was a 4MB Winbond 25Q32FVS1G memory chip. I traced it back to the STM32's Hardware SPI1 pins (PA4, PA5, PA6, PA7).

I wrote a quick reconnaissance scanner to ping the chip's "Driver's License" (the JEDEC ID). It responded with EF 40 16—loud and clear. I dumped Page Zero and immediately spotted 32-bit little-endian pointers. They acted as a hardcoded File Allocation Table, pointing deeper into the memory space.

This chip held the original fonts, animations, and the proprietary Chinese software config. I needed it.
6. The 4MB Heist (Extracting the Brains)

Dumping 4 Megabytes of binary data over a 115200 baud UART connection, which is then piped through an ESP32 TCP network bridge, is a recipe for disaster.

When I tried a simple Serial.write streaming loop, the network buffers overran, connection latency spiked, and bits dropped everywhere.

To reliably extract the data, I had to architect a custom Stop-and-Wait chunked protocol with built-in error correction:

    The STM32 (C++): Read a 4096-byte (4KB) chunk from the flash, calculate a strict CRC16-CCITT checksum, and send the 4098 bytes over the network.

    The Laptop (Python): Catch the chunk and run the exact same math. If the CRCs match, send an ACK. If the network corrupted a bit, send a NACK to force the STM32 to re-transmit that specific chunk.

```python
# The Self-Healing Python Catcher Logic
data = raw_chunk[:BLOCK_SIZE]
received_crc = (raw_chunk[BLOCK_SIZE] << 8) | raw_chunk[BLOCK_SIZE + 1]

calculated_crc = calculate_crc16(data)

if received_crc == calculated_crc:
    ser.write(b"ACK\n")
    dump_data.extend(data)
    block_accepted = True
else:
    print(f"\n[!] CRC Mismatch! Network drop detected. Sending NACK...")
    ser.reset_input_buffer()
    ser.write(b"NACK\n") # Forces STM32 to resend the exact 4KB block
```

After several minutes of blinking lights and a few successfully caught network drops, the script finished. I ran a full-file CRC32 check against the physical chip. It was a flawless 1:1 hardware clone.

## 7. Unmasking the Ghost in the Machine (Data Forensics)

Time to see what we stole. I ran a hexdump on the extracted .bin file.

Following the pointers from Page Zero, I landed at address 0x00005000. Right away, I found an ASCII manifest referencing yxled.prj, yxled.dat, and autime.cfg. A quick Google search confirmed this clock was originally programmed by "YXLED," a commercial display software suite.

The manifest showed the exact date the board was last flashed (2024-07-26 08:49:59), default IP/MAC addresses, and Chinese hardware identifiers encoded in standard GB2312 (translating to "Display Screen 1"). It also explicitly stated the size of the compiled graphics payload: 324 bytes.

But what about the graphics themselves?

I wrote a Python "ROM Ripper" using the Pillow (PIL) library. It takes the raw binary data and forcefully renders it as a 1-bit black-and-white image. By tweaking the pixel-width of the image generation, you can essentially visualize the raw hex code. Sure enough, scrolling through the static, the hidden bitmap fonts, UI elements, and default Chinese characters emerged perfectly. We owned the device.

## 8. Conclusion & Next Steps: Graduating from Arduino

We mapped the hardware, dumped the flash, and built a custom display multiplexer. But there is a glaring architectural problem moving forward.

The Arduino loop() architecture is fundamentally single-threaded and blocking. If I want to add a USB Host RS485 bridge to this clock (to connect it to the Project PRECINCT alarm board we discussed previously), the display multiplexing will stutter every time the USB stack triggers an interrupt.

To make this a true MekOps node, we need to graduate. In the next phase, we are ditching the Arduino framework entirely. We are porting Apache NuttX to the board to gain a POSIX-compliant Real-Time Operating System (RTOS).

Once we have proper thread scheduling and a virtual file system, we can finally tease the ultimate end-goal: sandboxing applications on a 64KB RAM chip using WebAssembly (Wasm).

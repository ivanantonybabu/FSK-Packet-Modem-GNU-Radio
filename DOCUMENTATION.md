# FSK Packet Modem — Detailed Documentation

## 1. Project Overview

This project implements a **packet-oriented binary FSK modem** using GNU Radio Companion. It is divided into two flowgraphs:

1. `FSKMODEMTX.grc` — transmitter
2. `FSKMODEMRX.grc` — receiver

The transmitter starts with a text file, converts the bytes into individual bits, expands each bit to the required number of samples, and maps the binary states to two different frequency values using a complex VCO. The resulting complex baseband signal is stored in a file named `data`.

The receiver reads the complex sample file, performs quadrature demodulation to convert frequency variation into amplitude/level information, shifts the resulting signal with an additive constant, slices it into binary values, reduces the sample rate, repacks the bits into bytes, and writes the recovered byte stream to `result.txt`.

The implementation is therefore a **file-based modem/testbed**. There is no UHD/USRP Source or Sink in either supplied GRC file.

---

# 2. Overall System Architecture

```text
                    TRANSMITTER

 input.txt
    │
    ▼
Repack Bits 8→1
    │
    ▼
Repeat each bit
    │
    ▼
Unsigned Char → Float
    │
    ▼
Amplitude scaling + VCO offset
    │
    ▼
Complex VCO
    │
    ▼
Throttle @ 48 kS/s
    ├──────────────► Qt GUI waveform
    │
    └──────────────► data (complex samples)


                     RECEIVER

 data (complex samples)
    │
    ├──────────────► Qt GUI input waveform
    │
    ▼
Quadrature Demodulator
    │
    ▼
Add Constant / level correction
    ├──────────────► Qt GUI demodulated signal
    │
    ▼
Binary Slicer
    │
    ├──────────────► Qt GUI recovered bits
    │
    ▼
Keep 1 in N
    │
    ▼
Repack Bits 1→8
    │
    ▼
Throttle @ 48 kS/s
    │
    ▼
result.txt
```

---

# 3. Main Design Parameters

Both flowgraphs use the following core FSK parameters.

| Parameter | Value | Purpose |
|---|---:|---|
| `samp_rate` | 48000 samples/s | Main baseband sample rate |
| `baud` | 1200 baud | Input bit rate |
| `mark` | 1200 Hz | One FSK frequency state |
| `space` | 2200 Hz | Other FSK frequency state |
| `center` | 1700 Hz | `(mark + space) / 2` |
| `fsk_deviation` | 1000 Hz | `abs(mark-space)` |
| `repeat` | 40 | Defined as `samp_rate/baud` in the supplied files |
| `sps` | 20 | Receiver variable: `repeat/decim` |
| `decim` | 2 | Receiver reduction factor |

The transmitter's VCO-related variables are:

```text
vco_max    = center + fsk_deviation
           = 1700 + 1000
           = 2700 Hz

vco_offset = space / vco_max
           ≈ 0.8148

inp_amp    = (mark / vco_max) - vco_offset
           ≈ -0.3704
```

However, the actual transmitter blocks contain fixed constants `0.06545` and `0.22253` rather than directly referencing the `inp_amp` and `vco_offset` variables. These values are therefore the effective block settings in the supplied GRC file.

---

# 4. Transmitter — `FSKMODEMTX.grc`

## 4.1 Purpose

The transmitter converts the contents of `input.txt` into a complex binary-FSK waveform. Each binary bit is represented by a different frequency. The generated complex waveform is saved as `data` for use by the receiver.

## 4.2 Transmitter Signal Chain

```text
input.txt
   ↓
File Source (byte)
   ↓
Repack Bits (8 → 1)
   ↓
Repeat
   ↓
UChar → Float
   ↓
Multiply Constant
   ↓
Add Constant
   ↓
Complex VCO
   ↓
Throttle
   ├──→ File Sink: data
   ├──→ Qt GUI Time Sink
   └──→ Complex → Real → Gain → Audio Sink
```

## 4.3 Block-by-Block Description

### 4.3.1 `blocks_file_source_0` — File Source

**Type:** `blocks_file_source`

Configuration:

- Input file: `I:\SKILLS\SDDR RADIO\PROJECT\gr-control-main\Transmitters\input.txt`
- Data type: byte
- Repeat: `True`

The block reads the source text file as a stream of bytes. Because repeat is enabled, the input can continuously circulate through the transmitter instead of stopping after one pass.

### 4.3.2 `blocks_repack_bits_bb_1_0` — Repack Bits

**Type:** `blocks_repack_bits_bb`

- Input width `k = 8`
- Output width `l = 1`
- Endianness: MSB first

A text file is naturally represented as bytes. FSK modulation in this project is performed at the bit level, so this block converts every 8-bit byte into individual binary symbols.

```text
Byte → b7 b6 b5 b4 b3 b2 b1 b0
```

### 4.3.3 `blocks_repeat_0` — Repeat

**Type:** `blocks_repeat`

The block repeats each binary symbol. The interpolation/repetition parameter is connected to the variable `repeat`, which is defined as:

```text
repeat = int(samp_rate / baud)
       = 48000 / 1200
       = 40
```

Thus, one logical bit is represented by 40 baseband samples.

### 4.3.4 `blocks_uchar_to_float_0` — UChar to Float

The repeated byte-valued binary stream is converted into floating-point values so that it can be processed by the amplitude-scaling and VCO stages.

### 4.3.5 `blocks_multiply_const_vxx_0` — Input Amplitude Scaling

**Type:** `blocks_multiply_const_vxx`

- Data type: float
- Effective constant: `0.06545`
- Comment: `inp_amp`

This scales the binary control signal before it is shifted by the following Add Constant block.

### 4.3.6 `blocks_add_const_vxx_0` — VCO Offset

**Type:** `blocks_add_const_vxx`

- Data type: float
- Effective constant: `0.22253`
- Comment: `vco_offset`

The scaled binary waveform is shifted to create the control range required by the VCO.

### 4.3.7 `blocks_vco_c_0` — Complex VCO

**Type:** `blocks_vco_c`

Parameters:

- Sample rate: `48000`
- Amplitude: `1.0`
- Sensitivity: `2*pi*vco_max`
- `vco_max = center + fsk_deviation`

The VCO converts the control voltage into a complex sinusoidal signal. Changes between the two binary levels cause the oscillator frequency to change, producing the FSK waveform.

Conceptually:

```text
Binary 0 → frequency state A
Binary 1 → frequency state B
```

The exact resulting frequency mapping depends on the effective constants used by the VCO control chain.

### 4.3.8 `blocks_throttle2_0` — Throttle

**Type:** `blocks_throttle2`

- Data type: complex
- Sample rate: `48000 samples/s`

The throttle controls processing speed in this file-based flowgraph so that the generated stream runs at the configured sample rate rather than being processed as fast as the host computer allows.

### 4.3.9 `blocks_file_sink_0` — Complex Sample File

**Type:** `blocks_file_sink`

- File: `data`
- Data type: complex
- Unbuffered: `True`

This stores the generated complex baseband samples. The receiver uses this file as its input.

### 4.3.10 `qtgui_time_sink_x_0` — Transmit Waveform Display

**Type:** Qt GUI Time Sink

- Input type: complex
- Samples displayed: 2048
- Sample rate: 48 kS/s
- Update time: 0.10 s
- Name: `Transmit_data`

This provides a visual representation of the generated complex FSK waveform in the time domain.

### 4.3.11 `blocks_complex_to_real_0`

This extracts the real component of the complex FSK signal.

### 4.3.12 `blocks_multiply_const_vxx_1`

Scales the real component by `0.3` before it is sent to the audio output.

### 4.3.13 `audio_sink_0`

Outputs the scaled real component through the system audio device at 48 kS/s. This is primarily a monitoring path and is not part of the digital FSK file-transfer path.

### 4.3.14 `virtual_sink_0`

**Stream ID:** `t2`

The transmitter sends the post-bit-conversion stream to a GNU Radio Virtual Sink. This can be used for an internal virtual-stream architecture, but in the supplied receiver the corresponding Virtual Source uses stream ID `r1`, not `t2`. Therefore these two flowgraphs are not directly connected through these virtual blocks.

---

# 5. Receiver — `FSKMODEMRX.grc`

## 5.1 Purpose

The receiver reconstructs binary data from the complex FSK samples stored in `data`.

Its main processing stages are:

```text
Complex FSK
   ↓
Quadrature Demodulation
   ↓
Level/offset adjustment
   ↓
Binary slicing
   ↓
Sample reduction
   ↓
1-bit → 8-bit repacking
   ↓
Output file
```

## 5.2 Block-by-Block Description

### 5.2.1 `blocks_file_source_0` — Received Complex Samples

**Type:** `blocks_file_source`

- File: `I:\SKILLS\SDDR RADIO\PROJECT\gr-control-main\Transmitters\data`
- Data type: complex
- Repeat: `False`

The receiver reads the complex baseband samples from the file generated by the transmitter.

### 5.2.2 `analog_quadrature_demod_cf_0` — Quadrature Demodulator

**Type:** `analog_quadrature_demod_cf`

Gain:

```text
gain = samp_rate / (2*pi*fsk_deviation)
```

With the configured values:

```text
gain = 48000 / (2*pi*1000)
     ≈ 7.64
```

The quadrature demodulator detects changes in the phase of the complex signal. Since instantaneous frequency is related to phase change, this converts the FSK frequency shifts into a real-valued signal that can be thresholded into binary states.

### 5.2.3 `blocks_add_const_vxx_0` — Demodulator Level Adjustment

**Type:** `blocks_add_const_vxx`

- Constant: `-0.7`
- Data type: float

This shifts the demodulator output before binary slicing. The offset helps place the two recovered FSK states around a suitable decision boundary.

### 5.2.4 `digital_binary_slicer_fb_0` — Binary Slicer

The binary slicer converts the floating-point demodulated waveform into binary values.

Conceptually:

```text
positive/above decision level → 1
negative/below decision level → 0
```

This is the main symbol-decision stage of the receiver.

### 5.2.5 `blocks_uchar_to_float_0_0` — Recovered Bit Visualization

This converts the recovered unsigned-byte bit stream into floating-point values for display in the Qt GUI time sink.

### 5.2.6 `blocks_keep_one_in_n_0` — Sample Reduction

**Type:** `blocks_keep_one_in_n`

- Data type: byte
- `N = 40`

Since the transmitter represents each bit using 40 samples, the receiver keeps one sample out of each group of 40. This reduces the oversampled binary stream back toward the original bit rate.

The receiver also defines:

```text
sps = repeat / decim
   = 40 / 2
   = 20
```

However, the actual `Keep One in N` block uses a fixed `N = 40` in the supplied GRC file.

### 5.2.7 `blocks_repack_bits_bb_1_0` — Repack Bits

**Type:** `blocks_repack_bits_bb`

- Input: 1 bit
- Output: 8 bits
- Endianness: MSB first

This reverses the transmitter's bit expansion process by grouping eight recovered binary symbols into one byte.

```text
1 0 0 1 0 1 1 0 → 10010110
```

The resulting bytes represent the recovered data.

### 5.2.8 `blocks_throttle2_0_0` — Output Rate Control

**Type:** `blocks_throttle2`

- Data type: byte
- Sample rate: `48000`
- Ignore tags: `True`

This controls processing speed on the recovered byte stream.

### 5.2.9 `blocks_file_sink_0` — Recovered Data

**Type:** `blocks_file_sink`

- Data type: byte
- File: `I:\SKILLS\SDDR RADIO\PROJECT\gr-control-main\Receivers\result.txt`
- Append: `False`
- Unbuffered: `True`

This is the final data output of the receiver. The recovered byte stream is written to `result.txt`.

---

# 6. Receiver Visualization Blocks

The receiver contains four Qt GUI time sinks.

## 6.1 `qtgui_time_sink_x_1` — Input/Testbed Signal

- Type: complex
- Size: 1024 samples
- Sample rate: 48 kS/s
- Name: `testbed`

It displays the complex samples read from the `data` file.

## 6.2 `qtgui_time_sink_x_1_0` — Float-to-Complex Testbed Path

- Type: complex
- Size: 1024 samples
- Sample rate: 48 kS/s
- Name: `testbed`

This displays the complex signal generated from the local `audio_source_0`/constant path rather than the main file-demodulation path.

## 6.3 `qtgui_time_sink_x_0` — Demodulated Signal

- Type: float
- Size: 1024 samples
- Sample rate: 48 kS/s

It displays the output after the quadrature demodulator and `-0.7` offset.

## 6.4 `qtgui_time_sink_x_0_2` — Correlate/Recovered-Bit Display

- Type: float
- Size: 128 samples
- Name: `Correlate input`
- GUI position: `2,0,1,3`
- Trigger level: `0.2`
- Trigger tag: `packet_len`

The block displays the recovered bit stream after binary slicing. The name `Correlate input` appears in the GRC configuration, although there is no explicit correlation block in the supplied receiver flowgraph.

---

# 7. Additional Receiver Blocks

### `audio_source_0`

An audio source configured at 48 kS/s. It is connected to `blocks_float_to_complex_0`, together with a zero-valued constant source. This forms a separate complex test/monitor path and is not connected to the main `data → demodulator → decoder` chain.

### `analog_const_source_x_0`

Generates a constant float value of `0`. It supplies the second input of the float-to-complex block.

### `blocks_float_to_complex_0`

Combines the audio-source float stream with the zero constant to create a complex signal. This signal is sent to the `testbed` GUI sink.

### `virtual_source_2`

**Stream ID:** `r1`

Supplies data to the binary slicer. Because the transmitter's Virtual Sink uses `t2`, this block does not receive the transmitter's virtual stream in the supplied pair of files.

---

# 8. Variables Present in the Receiver

| Variable | Value / Formula | Role |
|---|---|---|
| `baud` | 1200 | Nominal bit rate |
| `samp_rate` | 48000 | Processing sample rate |
| `mark` | 1200 | FSK frequency parameter |
| `space` | 2200 | FSK frequency parameter |
| `center` | `(mark+space)/2` | Center frequency parameter |
| `fsk_deviation` | `abs(mark-space)` | Frequency separation |
| `repeat` | `int(samp_rate/baud)` | Samples per bit from transmitter definition |
| `decim` | 2 | Reduction parameter |
| `sps` | `int(repeat/decim)` | Derived samples-per-symbol parameter |
| `phase_bw` | `pi/32` | Defined receiver parameter |
| `thresh` | 0 | Defined threshold parameter |
| `reverse` | Normal/Reverse chooser | GUI parameter |
| `sq_lvl` | -20 dB default | GUI squelch parameter |

Some of these variables are defined in the GRC file but are not connected to an active processing block in the supplied connection graph. They should therefore not be interpreted as active processing stages unless the flowgraph is modified.

---

# 9. Virtual Source/Sink Relationship

The two flowgraphs contain Virtual Source/Sink blocks:

| Flowgraph | Block | Stream ID |
|---|---|---|
| TX | `virtual_sink_0` | `t2` |
| RX | `virtual_source_2` | `r1` |

GNU Radio Virtual Source and Virtual Sink blocks communicate only when the corresponding stream identifiers match. Since `t2 ≠ r1`, these supplied blocks do not establish a direct TX-to-RX virtual connection.

The practical data path between the two flowgraphs is therefore the file:

```text
FSKMODEMTX.grc
       │
       ▼
      data
       │
       ▼
FSKMODEMRX.grc
```

---

# 10. Complete Implementation Procedure

## Step 1 — Prepare the input

Create a text file named `input.txt` containing the data to be transmitted.

The transmitter GRC currently contains a machine-specific path:

```text
I:\SKILLS\SDDR RADIO\PROJECT\gr-control-main\Transmitters\input.txt
```

For use on another computer, replace this path with the local input-file location.

## Step 2 — Run the transmitter

Open `FSKMODEMTX.grc` in GNU Radio Companion and execute it.

The flowgraph:

1. Reads the input bytes.
2. Converts bytes into individual bits.
3. Repeats each bit 40 times.
4. Converts the binary values into floating-point VCO control values.
5. Generates a complex FSK waveform.
6. Displays the waveform.
7. Writes complex samples to `data`.

## Step 3 — Verify the generated waveform

Use the `Transmit_data` Qt GUI time sink to observe the generated complex signal.

The waveform should contain transitions corresponding to changes between the two binary FSK states.

## Step 4 — Run the receiver

Open `FSKMODEMRX.grc` and execute it after the `data` file is available.

The receiver:

1. Reads complex samples from `data`.
2. Performs quadrature demodulation.
3. Applies the `-0.7` level correction.
4. Converts the waveform to binary values.
5. Keeps one sample from each 40-sample symbol interval.
6. Reconstructs 8-bit bytes.
7. Writes the recovered bytes to `result.txt`.

## Step 5 — Check the output

Compare `result.txt` with the original `input.txt` at the byte/content level.

A successful file-based test should produce recovered data corresponding to the transmitted bit stream, subject to correct alignment, parameter settings, and the actual contents of the generated `data` file.

---

# 11. Results and Observations

The supplied GRC files allow the following results to be directly established from the implementation:

### 11.1 FSK waveform generation

The transmitter produces a complex baseband waveform using a complex VCO controlled by the binary data stream.

### 11.2 FSK demodulation

The receiver uses a quadrature demodulator to convert frequency changes into a real-valued signal suitable for binary decisions.

### 11.3 Binary recovery

The binary slicer converts the demodulated waveform into a byte-valued binary stream.

### 11.4 Byte reconstruction

The receiver groups the recovered bits back into 8-bit bytes using MSB-first bit repacking.

### 11.5 File output

The recovered bytes are written to `result.txt`.

### 11.6 Visualization

The transmitter and receiver contain Qt GUI time sinks for inspecting the complex waveform and demodulated/recovered signals.

---

# 12. Results That Are Not Available From the GRC Files Alone

The flowgraph definitions do **not** contain measured experimental results for:

- Bit Error Rate (BER)
- Packet Error Rate (PER)
- SNR
- SINR
- Measured RF bandwidth
- RF output power
- Receiver sensitivity
- Communication range
- Noise performance
- Timing recovery error
- Frequency offset tolerance
- Measured end-to-end latency
- Exact percentage of successfully recovered bytes

These values should only be included in a project report if they are obtained from actual experiments or measurement logs.

---

# 13. Important Implementation Observations

## 13.1 This is not an RF/SDR hardware chain

Neither supplied GRC file contains:

- UHD USRP Source
- UHD USRP Sink
- RTL-SDR Source
- HackRF Source/Sink
- SoapySDR hardware block

The implementation is therefore a **software/file-based FSK modem testbed**.

## 13.2 Transmitter and receiver frequency variables

The project defines `mark = 1200 Hz`, `space = 2200 Hz`, `center = 1700 Hz`, and `fsk_deviation = 1000 Hz`. These values establish the intended FSK frequency relationship used by the VCO/demodulator design.

## 13.3 Receiver sample reduction

The transmitter defines 40 samples per bit. The receiver's active `Keep One in N` block also uses `N = 40`, which is consistent with recovering one decision sample per transmitted bit.

The receiver additionally defines `decim = 2` and `sps = 20`, but those variables are not used by the active `Keep One in N` block in the supplied flowgraph.

## 13.4 Packet terminology

The GRC descriptions call the project `packet FSK receive` and `packet FSK xmt`. However, the supplied flowgraphs do not contain a dedicated packet formatter, packet header generator, CRC block, access-code detector, or packet decoder. The implementation is therefore best understood from the actual block graph as a file-to-FSK and FSK-to-file bitstream modem testbed.

---

# 14. Block Summary

| Block | Flowgraph | Function |
|---|---|---|
| File Source | TX/RX | Reads source or complex sample file |
| Repack Bits | TX/RX | Converts bytes↔bits |
| Repeat | TX | Expands each bit to multiple samples |
| UChar to Float | TX | Converts binary values to float |
| Multiply Constant | TX | Scales VCO control signal |
| Add Constant | TX/RX | Applies level/offset correction |
| Complex VCO | TX | Generates complex FSK waveform |
| Throttle | TX/RX | Controls software processing rate |
| Complex to Real | TX | Extracts real signal component |
| Audio Sink | TX | Monitors real component |
| Quadrature Demod | RX | Converts phase/frequency variation to amplitude |
| Binary Slicer | RX | Makes binary symbol decisions |
| Keep One in N | RX | Reduces oversampled bit stream |
| Repack Bits | RX | Reconstructs bytes |
| File Sink | TX/RX | Stores generated/recovered data |
| Qt GUI Time Sink | TX/RX | Visualizes intermediate signals |
| Virtual Sink/Source | TX/RX | Internal GNU Radio stream mechanism |

---

# 15. File and Path Configuration

The supplied GRC files contain Windows-specific paths.

### Transmitter input

```text
I:\SKILLS\SDDR RADIO\PROJECT\gr-control-main\Transmitters\input.txt
```

### Transmitter complex output

```text
data
```

### Receiver complex input

```text
I:\SKILLS\SDDR RADIO\PROJECT\gr-control-main\Transmitters\data
```

### Receiver output

```text
I:\SKILLS\SDDR RADIO\PROJECT\gr-control-main\Receivers\result.txt
```

These paths must be updated when moving the project to another computer.

---

# 16. Software Requirements

The flowgraphs are GNU Radio Companion files and specify:

- GNU Radio / GNU Radio Companion
- Qt GUI support
- Python output generation

The supplied files were generated with:

```text
GRC version: 3.10.12.0
```

The exact compatibility of individual blocks can depend on the installed GNU Radio version.

---

# 17. Recommended Test Setup

For a clean software test:

```text
             ┌─────────────────────┐
             │     input.txt       │
             └──────────┬──────────┘
                        │
                        ▼
                ┌──────────────┐
                │ FSKMODEMTX   │
                └──────┬───────┘
                       │
                       ▼
                    data
                       │
                       ▼
                ┌──────────────┐
                │ FSKMODEMRX   │
                └──────┬───────┘
                       │
                       ▼
                  result.txt
```

The most direct validation is to compare the original input data with the recovered output after the transmitter and receiver have completed processing.

---

# 18. Possible Extension to a Real SDR

The current project can serve as a base for an RF FSK modem by replacing the file interface with SDR hardware blocks.

A conceptual hardware version would be:

```text
Input Data
   ↓
FSK Modulator
   ↓
SDR Transmitter
   )))))) RF (((((
SDR Receiver
   ↓
FSK Demodulator
   ↓
Recovered Data
```

Such an extension would require additional considerations including RF center frequency, sample-rate compatibility, transmit/receive gain, filtering, synchronization, frequency offset, antenna configuration, and regulatory requirements.

---

# 19. Conclusion

The project demonstrates the complete software processing chain of a binary FSK modem. `FSKMODEMTX.grc` converts text bytes into a complex FSK waveform and stores the samples in `data`. `FSKMODEMRX.grc` reads those samples, performs quadrature frequency demodulation, makes binary decisions, reconstructs bytes, and stores the result in `result.txt`.

The flowgraphs also provide useful time-domain visualization points, making the project suitable for studying the relationship between digital data, FSK modulation, complex baseband signals, frequency/phase demodulation, binary slicing, and byte reconstruction.

The documentation above intentionally distinguishes **what is implemented in the supplied GRC files** from performance results that would require actual experimental measurements.

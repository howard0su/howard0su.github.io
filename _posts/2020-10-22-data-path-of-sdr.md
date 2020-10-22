---
layout: post
title:  "The Signal Path of Teensy Convolution SDR"
date:   2020-09-28 10:14:00
categories: [ SDR ]
featured: true
---

Teensy Convolution SDR is a great project. The signal path is complex but I feel it is worth to dig out more details of it.

### Stage1

#### Input:

Get I2S data from code through DMA.

#### Output:

32 Blocks, with each block 128 int16_t. In total, 4096 samples, 8KB x 2 channels.

### Stage2: Normalized to Float

#### Output:

32 Blocks, with each block 128 float32_t between -1.0f to 1.0f. In total, 4096 samples, 16KB x 2 channels.

#### Notes:

1. There is a proceduce to limit bitnumber, which force 0 of N LSBs. Not sure what this is for.
2. CMSIS DSP provides a SIMD API arm_q15_to_float to normalize to float.

### Stage3: IQ Balance Fix

IQ Balance fix contains two steps, which change the scale and the phase. There are three different algorithm implemented:

 1. Manual, the user manually config IQ_amplitude_correction_factor and IQ_phase_correction_factor.

 2. Mosely Algorithm:
```
Moseley, N.A. & C.H. Slump (2006): A low-complexity feed-forward I/Q imbalance compensation algorithm. http://doc.utwente.nl/66726/1/moseley.pdf
```
4. Chang Algorithm:
```
IQ imbalance correction algorithm by Chang et al. 2010
```

### Stage4: Move center freq from DC to +FS/4
Use a fast algorithm here,

```
Frequency translation by Fs/4 without multiplication
Lyons (2011): chapter 13.1.2 page 646
```

### Stage5: Caculate Spectrum，dbm

If spectrum zoom is 1, use the first 256 samples before stage 4.

### Stage6: 8x decimate

First do a 4x decimate, then a 2x. The bandsiwth changes to 96/8=12khz.
CMSIS DSP API is arm_fir_decimate_f32

#### Output:
32 Blocks，Each blocks is 16 float32, total 512 samples. 

### Stage7: Convolution
Merge IQ into a complex_t and do the FFT convolution. 
1. Use last loop data, merge with the current set of data. In totally 1024 samples, each sample is a complex data (im, re).
2. FFT, 1024 bins in frequence domain

### Stage8：Autotune
Find the highest power bins （I^2+Q^2), so the baseband can be determined.

```
Lyons (2011): chapter 13.15 page 702
```

### Stage9: Supress single band
If we are using SSB, supress single band.

```
"frequency translation without multiplication" - DSP trick R. Lyons (2011))
```

### Stage10：FIR Filter
Band Pass Filter in FIR.
CMSIS DSP API is arm_cmplx_mult_cmplx_f32

The filter is created when swiching the band:
calc_cplx_FIR_coeffs (FIR_Coef_I, FIR_Coef_Q, m_NumTaps, (float32_t)bands[current_band].FLoCut, (float32_t)bands[current_band].FHiCut, (float)SR[SAMPLE_RATE].rate / DF);

### Stage11: Notch Filter
Based on notch setting (center frequence and width), clear the corresponding bins to 0.

### Stage12：iFFT
iFFT to get the data back to time domain.
CMSIS DSP API is arm_cfft_f32.

### Stage13: Gain Control
There are two algorithm here:
1. Manual GC，Gain is a user configuration.
2. AGC，Gain is caculated based on the current samples.
Use Gain as a scale factor to multiply each sample.

### Stage14； Demodulize 
This is different for different mode. Check the next section for details.

#### 输出
256 float32 data for audio, Left and Right, two channels.

### Stage15：Audio EQ
Based on EQ settings, generate two FIR filter, and apply filter to the audio data of two channels.
CMSIS DSP API is arm_fir_f32.

### Stage16: LMS NR

```
variable-leak LMS algorithm
can be switched to NOISE REDUCTION or AUTOMATIC NOTCH FILTER
only one channel --> float_buffer_L --> NR --> float_buffer_R
```

### Stage17: Noise Blanker
Michael Wild's algorithm，noise Blanker

### Stage18:　Digit mode decode
Will decode CW, RTTY and DCF77.

### Stage19: interpolate
8x interpolate to make audio data suitable to play. and also scale the data based on volume control.

CMSIS DSP API is arm_fir_interpolate_f32 and arm_scale_f32。

### Stage20: convert to int16
Audio hardware requires int16, convert it here.
CMSIS DSP API is arm_float_to_q15.



## Demodule Details

### AM
### SAM
### LSB
### USB
### CW
### NFM
### WFM

## Digit Decode Details

### CW
### RTTY
### DCF77


## Some caculations

### How much memory is used for memory

I2S input：4096 samples X int16 X [I.Q]

float input: 4096 samples X float32 X [I, Q]

FFT buffer; 1024 Samples X float 32 X [im, re]

IFFT buffer: 1024 Samples X float 32 X [im, re]

audio float buffer：256 Samples X float 32 X [L, R]

audio output：4096 samples X int16 X [L, R]

Float input and audio float buffer can be shared, IFFT and FFT can be shared but the current implementation doesn't.

### How quick for each loop

The codec is running at single channel 96Khz. In order to get 32x512blocks, it took about 4/96=41ms. So each loop is about 41ms, which implies 24 loops/s.


### Future Optimizations

#### Some FIR fitler can be static so we can save some RAM.

#### Initialize float array to 0.0 can use memset(0) instead. zero in int and in float are same, all 0 in binary.

#### Append the previous block logic can be simplied.

#### FFT and iFFT can reuse the buffer.

#### Data normalize can be done for each block, instead of 8 blocks at a time.




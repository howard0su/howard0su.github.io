---
layout: post
title:  "The Signal Path of Teensy Convolution SDR"
date:   2020-09-28 10:14:00
categories: [ PlutoSDR, SDR ]
featured: false
---

Teensy Convolution SDR is a great project. The signal path is complex but I feel it is worth to dig out more details of it.

### 第一步

#### 输入:

从DMA里拿到的I2S的数据，

#### 输出:

32块，每一块是128个16位的有符号整数。一共4096个数据，一共8KB X 2

### Stage2: 归一到浮点数

#### 输出:

32块，每一块是128个16位的-1.0到1.0的浮点数。一共4096个数据，一共16KB X 2

#### 处理:

1. 有一个bitnumber的处理，就是把低位的数据清0，降低ADC在LSB的干扰。
2. arm加速的SIMD版本的API是arm_q15_to_float，同时归一化和转化成浮点数。

### Stage3: IQ平衡矫正

IQ 矫正是有两个步骤，不改变数据的格式和大小。分别矫正IQ的幅度（scale）和相位（phase）。这里有三个算法：

 1. 手工，定义IQ_amplitude_correction_factor和IQ_phase_correction_factor去校准数据。

 2. Mosely算法:
```
Moseley, N.A. & C.H. Slump (2006): A low-complexity feed-forward I/Q imbalance compensation algorithm. http://doc.utwente.nl/66726/1/moseley.pdf
```
4. Chang算法:
```
IQ imbalance correction algorithm by Chang et al. 2010
```

### Stage4: FS/4的频率转换
使用一个不用乘法的算法。(有必要吗？乘法也是1个cycle）。移动中央频率到fs/4的地方。

```
Frequency translation by Fs/4 without multiplication
Lyons (2011): chapter 13.1.2 page 646
```

### Stage5: 计算频谱，dbm

这里用了一个优化，如果zoom=1，在频率移动之前的256个sample来计算。（这样的优化没有意义，zoom=1不是常见路径，省出来的CPU也不能做其他的事情）

其实没有需要每个loop都记算频谱，完全可以若干次数据算一次的。

### Stage6: 8倍抽取

先做一次4倍的抽取，再做一次2倍的抽取。一共8倍的抽取，采样带宽变成96/8=12khz。
arm加速的SIMD版本的API是arm_fir_decimate_f32.

#### 输出
32块，每一块是16个浮点数的结果。总共512个数据。

### Stage7: Convolution
将把I，Q数据组成Complex的float组，进行FFT。步骤如下：
1. 把上一个周期的数据和当前的数据拼成一个，总共1024个数据点。每个数据点2个浮点数。
2. 进行FFT，快速傅立叶变换，得到一个1024个数据点。
3. 得到了一个在频域的数据

### Stage8：Autotune
寻找最强的数据点，从频域数据中计算每个频点的信号强度（I^2+Q^2)。最强的频点就是需要切换的频率。
这个步骤可以考率只在用户调节过频率以后进行。

```
Lyons (2011): chapter 13.15 page 702
```

### Stage9: 去除单边带，如果是解码单边带数据
去除单边带数据，如果数据是LSB或者USB，在这个步骤去掉一个边带。
当前代码是注释掉的，需要研究为什么。

```
"frequency translation without multiplication" - DSP trick R. Lyons (2011))
```

### Stage10：FIR Filter
使用arm_cmplx_mult_cmplx_f32，作为filter。

此filter实在初始化时候建立的，和当前band和采样率有关的一个函数。
calc_cplx_FIR_coeffs (FIR_Coef_I, FIR_Coef_Q, m_NumTaps, (float32_t)bands[current_band].FLoCut, (float32_t)bands[current_band].FHiCut, (float)SR[SAMPLE_RATE].rate / DF);

### Stage11: Notch Filter
根据notch的宽度和值，在FFT中对应的项设为0. 

### Stage12：iFFT
将频域的数据重新转化成时域的数据。arm加速的SIMD版本的API是arm_cfft_f32。

### Stage13: AGC
自动Gain控制。AGC有多种模式；
1. 手工GC，Gain是一个用户设定的值。
2. 自动GC，Gain是一个计算的过程。
最后用Gain去放大每一个sample

### Stage14；解码
这一个步骤是对每一种编码模式都不同的。具体解码过程下面分开讨论。每一种的都不同。

#### 输出
256个Audio的数据，在左右两个通道。

### Stage15：声音EQ
根据EQ设置，生成两个FIR filter。将其filter两路左右声音数据。
arm加速的SIMD版本的API是arm_fir_f32

### Stage16: LMS 处理

```
variable-leak LMS algorithm
can be switched to NOISE REDUCTION or AUTOMATIC NOTCH FILTER
only one channel --> float_buffer_L --> NR --> float_buffer_R
```

### Stage17: Noise Blanker
Michael Wild的算法，去noise

### Stage18:　数字模式解码
这里开始解码CW，RTTY和DCF77的数据。可以增加其他的算法比如D-Star，或者DMR。

### Stage19: 插值
做2倍的插值，再做4倍的插值，总共8倍的抽取。并进行声音的放大和音量控制。arm的加速API是arm_fir_interpolate_f32和arm_scale_f32。

### Stage20: 声音数据产生
声音数据需要是16位的整型数字，现在开始转换。arm的加速API是arm_fir_interpolate_f32和arm_float_to_q15。



## Demod的算法细节

### AM
### SAM
### LSB
### USB
### CW
### NFM
### WFM

## 数字模式的算法细节

### CW
### RTTY
### DCF77



## 计算的数字

### 内存一共用了多少？

输入内存：4096 samples X int16 X [I.Q]

浮点内存: 4096 samples X float32 X [I, Q]

FFT内存; 1024 Samples X float 32 X [im, re]

IFFT内存: 1024 Samples X float 32 X [im, re]

声音内存：256 Samples X float 32 X [L, R]

输出内存：4096 samples X int16 X [L, R]

其中声音内存和浮点内存共享，IFFT和FFT内存可以共享（现在代码没有）

### 每个循环要多快：

96K，采集4K数据需要4/96=41ms。每个循环需要在41ms内结束，也就是24frame/s。同时这个速度可以是UI的刷新速度。

### 为什么需要抽取和插值都做2次，不一次完成？

FIR的filter效率更好。

### 优化点：所有的抽取和插值的数据结构都可以静态初始化，节省内存。

### 优化点：初始化float数组为0，可以用memset，float的0和int32的0是一样的。

### 优化点：第一个block的逻辑可以通过预先初始化保存的块来完成

### 优化点：IFFT和FFT的内存复用

### 优化点：数据的归一化可以每次都做，不用等到8个块到达

### 优化点：FIR filter的数据可以是静态初始化的，节省RAM





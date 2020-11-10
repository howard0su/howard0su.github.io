---

layout: post
title:  "Elecraft KX3 deep dive"
date:   2020-11-10 22:14:00
categories: [ SDR ]
featured: true

---

## Elecraft KX3 deep dive 他山之石，可以攻玉

This serial of blogs are analyzing the details of the schematic of KX3 for my personal study purpose.

### RX analog path overview

Receiver is getting the signal from antenna and feed into a Elliptic LPF network. The LPF network is using 7 relays to control the network. The first stage of input is a 6m band LPF (50Mhz), the second stage is selecting one of 6 LPFs.

```mermaid
graph LR
RF(Roof Filter)
LPF_Network[[LPF Network with 6 bands]]
BPF_Network[[BPF Network with 6 bands]]
Preamp_10db(Amp +10db with NE46134)
ATT_15db(Attenuator -15db with Resistor)
AMP_10db(LNA +10db with AD8099)
Mixer(Quadrature Mixer with 2 3257)

AF_AMP[AD8599 +15db]
Post_AMP[FSA1259A +5/+20db]
ADC[PCM1808 2ch 24bit]

Mixer ==> IQ_OUTPUT_Ext[External IQ output]

Attena --> LPF_Network --> HPF_1.8Mhz --> BPF_Network --> Preamp_10db --> ATT_15db --> AMP_10db --> convert_2_ends ==> Mixer ==> AF_AMP ==> RF ==> Post_AMP ==> ADC


```

#### Low Pass Filter Network and High Pass Filter

6m Band is 2 holes filter, and other LPFs are 2 - 3 holes. High Pass filter is one hole.

```mermaid
graph TD
Input --> LPF_50M
LPF_50M --> LPF_30M
LPF_50M --> LPF_20M
LPF_50M --> LPF_14M
LPF_50M --> LPF_7M
LPF_50M --> LPF_3.5M
LPF_50M --> LPF_1.8M
LPF_50M --> Bypass

LPF_30M --> HPF_1.8Mhz
LPF_20M --> HPF_1.8Mhz
LPF_14M --> HPF_1.8Mhz
LPF_7M --> HPF_1.8Mhz
LPF_3.5M --> HPF_1.8Mhz
LPF_1.8M --> HPF_1.8Mhz
Bypass --> HPF_1.8Mhz

```

The relays are controlled via a GPIO extension chip MCP23S17.

#### Band Pass Filter Network



### Clock Generation for ADC/DAC






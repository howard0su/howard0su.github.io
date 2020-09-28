---
layout: post
title:  "Port Dump1090 to PlutoSDR with IIO"
date:   2020-09-28 10:14:00
categories: [ PlutoSDR, SDR ]
featured: true
---
Network performance is still under investigation. The discouraging part is that the performance can only reach to 90M/s even with MSG_ZEROCOPY with new iperf3.

I am feeling that better approach should be processing the data more on the A9 CPUs on the board instead of transferring the data to PC. One of example is that KiwiSDR architecture, which processes the high bandwidth data to a audio I/Q stream (10k bandwith) and a waterfall data (8k bandwidth).

So I decided to add some small applications to the firmware to try. The first application is dump1090, which already in buildroot/package. However the version in the buildroot is only support rtlsdr. So I decided to do a small porting work to make it working via integrate two repos: https://github.com/MalcolmRobb/dump1090 and https://github.com/PlutoSDR/dump1090.

The code change is relative simple, which costs me a morning to finish the porting. dump1090 now works both from your PC or running from PlutoSDR directly.

```
Hex     Mode  Sqwk  Flight   Alt    Spd  Hdg    Lat      Long   Sig  Msgs   Ti|
-------------------------------------------------------------------------------
7813B4  S                    27825                                8    16    5
89901B  S                           261  355                     11    16    5
7808FD  S                           268  313                     20    16   13
780EBC  S                    18725  358  352                     28    32   19
7BB0A4  S                    18200                                7    16   49
780B1A  S     3036  CHH7251  33100  524  115   31.419  120.997   12   192    7
780DE2  S           CES5343   1725  185  094                     21   256    0
78157F  S                     5825                               14    32   36
78076F  S     1043  CCA4592  24150  392  336                      7   208    7
```

When running from PlutoSDR directly, the CPU consumption is low (10% with 2 CPUs, 20% with 1 CPU): 

```
Mem: 72652K used, 437016K free, 84K shrd, 0K buff, 45528K cached
CPU:   9% usr   6% sys   0% nic  84% idle   0% io   0% irq   0% sirq
Load average: 0.32 0.18 0.06 3/73 1803
  PID  PPID USER     STAT   VSZ %VSZ %CPU COMMAND
 1544  1511 root     S    33540   7%  10% dump1090 --interactive
  808     1 root     S    50964  10%   5% /usr/sbin/iiod -D -n 3 -F /dev/iio_ffs
 1610  1606 root     R     2692   1%   0% top
 1500   976 root     S     2608   1%   0% /usr/sbin/dropbear -R
 1599   976 root     S     2608   1%   0% /usr/sbin/dropbear -R
  968     1 avahi    S     3384   1%   0% avahi-daemon: running [pluto.local]
```

Checkout the code here: [https://github.com/howard0su/dump1090](https://github.com/howard0su/dump1090)
---
layout: post
title:  "Raw network performance of PlutoSDR"
date:   2020-09-22 22:01:00
categories: [ PlutoSDR, SDR ]
featured: true
---

In order to better understand what the raw network performance is on PlutoSDR, I decided to use iperf to run some benchmark.

I modified the defconfig in buildroot to include iperf tool in the firmware. 
```
+BR2_PACKAGE_IPERF=y
```
Then I run the following command on PlutoSDR to make it run as the server:
```
iperf -s -p 12345 -i 1 -M 
```

Then I runs several commands on my PC acting as the client to get the results:
```
$ iperf -c 169.254.2.35 -p 12345 -i 1 -t 5 -w 16K
------------------------------------------------------------
Client connecting to 169.254.2.35, TCP port 12345
TCP window size: 32.0 KByte (WARNING: requested 16.0 KByte)
------------------------------------------------------------
[  3] local 169.254.2.53 port 34838 connected with 169.254.2.35 port 12345
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 1.0 sec  54.9 MBytes   460 Mbits/sec
[  3]  1.0- 2.0 sec  56.4 MBytes   473 Mbits/sec
[  3]  2.0- 3.0 sec  57.0 MBytes   478 Mbits/sec
[  3]  3.0- 4.0 sec  56.2 MBytes   472 Mbits/sec
[  3]  4.0- 5.0 sec  56.0 MBytes   470 Mbits/sec
[  3]  0.0- 5.0 sec   280 MBytes   470 Mbits/sec
```
Then I uses the server side TCP window size test again, I got the best result.
```
$ iperf -c 169.254.2.35 -p 12345 -i 1 -t 5 -w 80K
------------------------------------------------------------
Client connecting to 169.254.2.35, TCP port 12345
TCP window size:  160 KByte (WARNING: requested 80.0 KByte)
------------------------------------------------------------
[  3] local 169.254.2.53 port 34862 connected with 169.254.2.35 port 12345
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 1.0 sec  71.9 MBytes   603 Mbits/sec
[  3]  1.0- 2.0 sec  72.9 MBytes   611 Mbits/sec
[  3]  2.0- 3.0 sec  72.9 MBytes   611 Mbits/sec
[  3]  3.0- 4.0 sec  73.6 MBytes   618 Mbits/sec
[  3]  4.0- 5.0 sec  72.1 MBytes   605 Mbits/sec
[  3]  0.0- 5.0 sec   363 MBytes   609 Mbits/sec
```

Let's further increase the package size to see how much we can get:

```
$ iperf -c 169.254.2.35 -p 12345 -i 1 -t 5 -w 200K
------------------------------------------------------------
Client connecting to 169.254.2.35, TCP port 12345
TCP window size:  400 KByte (WARNING: requested  200 KByte)
------------------------------------------------------------
[  3] local 169.254.2.53 port 34842 connected with 169.254.2.35 port 12345
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 1.0 sec  72.1 MBytes   605 Mbits/sec
[  3]  1.0- 2.0 sec  71.4 MBytes   599 Mbits/sec
[  3]  2.0- 3.0 sec  71.4 MBytes   599 Mbits/sec
[  3]  3.0- 4.0 sec  71.5 MBytes   600 Mbits/sec
[  3]  4.0- 5.0 sec  71.5 MBytes   600 Mbits/sec
[  3]  0.0- 5.0 sec   358 MBytes   600 Mbits/sec
```
Last try, let us enable the second core:
```
$ iperf -c 169.254.2.35 -p 12345 -i 1 -t 5 -w 80K
------------------------------------------------------------
Client connecting to 169.254.2.35, TCP port 12345
TCP window size:  160 KByte (WARNING: requested 80.0 KByte)
------------------------------------------------------------
[  3] local 169.254.2.53 port 35668 connected with 169.254.2.35 port 12345
[ ID] Interval       Transfer     Bandwidth
[  3]  0.0- 1.0 sec  85.9 MBytes   720 Mbits/sec
[  3]  1.0- 2.0 sec  85.8 MBytes   719 Mbits/sec
[  3]  2.0- 3.0 sec  85.1 MBytes   714 Mbits/sec
[  3]  3.0- 4.0 sec  86.0 MBytes   721 Mbits/sec
[  3]  4.0- 5.0 sec  85.5 MBytes   717 Mbits/sec
[  3]  0.0- 5.0 sec   428 MBytes   718 Mbits/sec
```

So the current hardware with the current Linux driver, 85MB/s is maxium throughput. Let's do some caculation. Each sample has two int16 I & Q. So total is 4 bytes. 85MB / 4 = 21Msps. So in theory, we can only support up to 21Msps bandwidth.

Is this good enough? Maybe I should take a look at the network driver to see if we can get. 
---

layout: post
title:  "How to debug Pico with JLINK OB under Windows"
date:   2023-01-22 20:14:00
categories: [ Pico, SDR ]
featured: true
published: true

---

I were using a RPI4 to write the code for Pico. It is additional effort and table space waste to have another board. So I decided to setup the development enviroment on Windows with WSL2 this afternoon. This post captured the issues and the solutions.

## Enviroment:
- Software: Windows 10, with WSL2 installed. Ubuntu 20.04 LTS. VSCODE with Cortex-Debug
- Hardware: RPI Pico, JLINK OB

## Steps in high level:
1. Install rp2040 openocd to replace the ubuntu default one
2. Install necessary vscode extensions like corext-debug

## Issue 1: Cannot run openocd as normal user
The solution is simple to modify udev scipt. The pullze is that you have to start udev service.

## Issue 2: Corext config is hard to get right.
Here is the copy of my working config:
```
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Pico Debug",
            "cwd": "${workspaceRoot}",
            "executable": "${command:cmake.launchTargetPath}",
            "request": "launch",
            "type": "cortex-debug",
            "servertype": "openocd",
            // This may need to be arm-none-eabi-gdb depending on your system
            "gdbPath" : "gdb-multiarch",
            "device": "RP2040",
            "configFiles": [
                "interface/jlink-swd.cfg",
                "target/rp2040.cfg"
            ],
            "rtos": "FreeRTOS",
            "interface": "swd",
            "svdFile": "../pico-sdk/src/rp2040/hardware_regs/rp2040.svd",
            "rttConfig":{
                "enabled": true,
                "address": "auto",
                "clearSearch": true,
                "decoders": [
                    {
                        "port": 0,
                        "type": "console"
                    }
                ]
            }
        }
    ]
}
```

## Issue 3: nm-multiarch and obj-multiarch doesn't exist
Even I installed "apt install binutils-multiarch", the commands still cannot find.
```
cd /usr/bin/
sudo ln -s x86_64-linux-gnu-nm nm-multiarch
sudo ln -s x86_64-linux-gnu-objdump objdump-multiarch
```

## Issue 4: corext-Debug fails with jtag is used while I am using swd
This is due to the jlink script is default to jtag. I created jlink-swd.cfg under /usr/share/openocd/scripts/interface as following:
```
adapter driver jlink
transport select swd
adapter speed 1000
```

After the settings, everything works well including RTT debug output. Nice!


# 基于Ubuntu进行STM32程序开发

## platform info

> Ubuntu version : Ubuntu 20.04.1 LTS
> 正点原子 STM32f767 阿波罗开发板
> st-link version : v1.6.1-201-g4bfaab0
> openocd version : 0.10.0
> Cross Compiler :  gcc-arm-none-eabi (9.2.1 20191025 release)

## install stlink driver

```shell
sudo apt install libusb-1.0-0-dev gcc build-essential cmake libusb-1.0-0

git clone https://github.com/texane/stlink.git

cd stlink
make clean 
make release
make debug

sudo cp -a  config/udev/rules.d/* /etc/udev/rules.d/
sudo udevadm  control  --reload-rules
sudo udevadm  trigger

cd build/Release/bin/
echo "export PATH=\$PATH:`pwd` " >> ~/.bashrc 
source ~/.bashrc
```

### test 

```shell
$ st-info  --probe
Found 1 stlink programmers
 serial:     323f68064248323848522157
 hla-serial: "\x32\x3f\x68\x06\x42\x48\x32\x38\x48\x52\x21\x57"
 flash:      2097152 (pagesize: 2048)
 sram:       524288
 chipid:     0x0451
 descr:      F76xxx
```

## openocd

```shell
#git clone https://github.com/ntfreak/openocd.git
sudo apt install openocd
```

### test 

将st-link和开发板相连，并给板子上电。

```shell
$ openocd -f /usr/share/openocd/scripts/interface/stlink-v2.cfg  -f /usr/share/openocd/scripts/target/stm32f7x.cfg 
Open On-Chip Debugger 0.10.0
Licensed under GNU GPL v2
For bug reports, read
	http://openocd.org/doc/doxygen/bugs.html
Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
adapter speed: 2000 kHz
adapter_nsrst_delay: 100
srst_only separate srst_nogate srst_open_drain connect_deassert_srst
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
Info : Unable to match requested speed 2000 kHz, using 1800 kHz
Info : clock speed 1800 kHz
Info : STLINK v2 JTAG v35 API v2 SWIM v7 VID 0x0483 PID 0x3748
Info : using stlink api v2
Info : Target voltage: 3.230647
Info : stm32f7x.cpu: hardware has 8 breakpoints, 4 watchpoints
```


## 创建工程

可以直接使用 `stm32CubeMX` 生成 `Makefile` 工程 。

```
.
├── Core
│   ├── Inc
│   │   ├── gpio.h
│   │   ├── main.h
│   │   ├── stm32f7xx_hal_conf.h
│   │   └── stm32f7xx_it.h
│   └── Src
│       ├── gpio.c
│       ├── main.c
│       ├── stm32f7xx_hal_msp.c
│       ├── stm32f7xx_it.c
│       └── system_stm32f7xx.c
├── Drivers
│   ├── CMSIS
│   │   ├── Device
│   │   └── Include
│   └── STM32F7xx_HAL_Driver
│       ├── Inc
│       └── Src
├── led.ioc
├── Makefile
├── startup_stm32f767xx.s
└── STM32F767IGTx_FLASH.ld

其中 STM32F7xx_HAL_Driver 存储的是HAL的代码，此处不再展开。
CMSIS的内容如下
./Drivers/CMSIS/
├── Device
│   └── ST
│       └── STM32F7xx
│           └── Include
│               ├── stm32f767xx.h
│               ├── stm32f7xx.h
│               └── system_stm32f7xx.h
└── Include
    ├── cmsis_compiler.h
    ├── cmsis_gcc.h
    ├── cmsis_version.h
    ├── core_cm7.h
    └── mpu_armv7.h
```

中最重要的也是和Keil中不一样的地方，就是以下三个部分 

- `Makefile` 
- 链接器脚本 `STM32F767IGTx_FLASH.ld` 
- 启动文件 `startup_stm32f767xx.s`

内容都比较简单，此处不列出来了。

## 编译代码

```shell
$ sudo apt install gcc-arm-none-eabi

$ cd led # 进入到工程目录
$ make
```

## 烧写代码

1. 打开openocd服务

	打开一个Terminal窗口输入以下命令后回车
	```shell
	openocd -f /usr/share/openocd/scripts/interface/stlink-v2.cfg  -f /usr/share/openocd/scripts/target/stm32f7x.cfg
	```

	执行过程如下
	
	```shell
	$ openocd -f /usr/share/openocd/scripts/interface/stlink-v2.cfg  -f /usr/share/openocd/scripts/target/stm32f7x.cfg  
	Open On-Chip Debugger 0.10.0
	Licensed under GNU GPL v2
	For bug reports, read
		http://openocd.org/doc/doxygen/bugs.html
	Info : auto-selecting first available session transport "hla_swd". To override use 'transport select <transport>'.
	Info : The selected transport took over low-level target control. The results might differ compared to plain JTAG/SWD
	adapter speed: 2000 kHz
	adapter_nsrst_delay: 100
	srst_only separate srst_nogate srst_open_drain connect_deassert_srst
	Info : Unable to match requested speed 2000 kHz, using 1800 kHz
	Info : Unable to match requested speed 2000 kHz, using 1800 kHz
	Info : clock speed 1800 kHz
	Info : STLINK v2 JTAG v35 API v2 SWIM v7 VID 0x0483 PID 0x3748
	Info : using stlink api v2
	Info : Target voltage: 3.239128
	Info : stm32f7x.cpu: hardware has 8 breakpoints, 4 watchpoints
	# --------------------------------------  下边的内容是执行客户端指令时的输出
	Info : accepting 'telnet' connection on tcp/4444
	target halted due to debug-request, current mode: Thread 
	xPSR: 0x81000000 pc: 0x08001142 msp: 0x2007ffe8
	target halted due to debug-request, current mode: Thread 
	xPSR: 0x01000000 pc: 0x0800116c msp: 0x20080000
	** Programming Started **
	auto erase enabled
	Info : device id = 0x10016451
	Info : flash size = 1024kbytes
	Info : Single Bank 1024 kiB STM32F76x/77x found
	target halted due to breakpoint, current mode: Thread 
	xPSR: 0x61000000 pc: 0x20000046 msp: 0x20080000
	wrote 32768 bytes from file build/led.bin in 2.035821s (15.718 KiB/s)
	** Programming Finished **
	** Verify Started **
	target halted due to breakpoint, current mode: Thread 
	xPSR: 0x61000000 pc: 0x2000002e msp: 0x20080000
	verified 4612 bytes in 0.716718s (6.284 KiB/s)
	** Verified OK **
	shutdown command invoked
	Info : dropped 'telnet' connection
	
	```


2. telnet 到 openocd 服务

	再打开一个Terminal窗口，输入 `telnet localhost 4444` , 之后在提示符后输入下边指令

	```shell
	> halt
	> program build/led.bin 0x8000000 verify
	> shutdown
	```

	执行过程如下：

	```shell
	$ telnet localhost 4444
	Trying 127.0.0.1...
	Connected to localhost.
	Escape character is '^]'.
	Open On-Chip Debugger
	> halt
	target halted due to debug-request, current mode: Thread 
	xPSR: 0x81000000 pc: 0x08001142 msp: 0x2007ffe8
	> program build/led.bin 0x8000000 verify
	target halted due to debug-request, current mode: Thread 
	xPSR: 0x01000000 pc: 0x0800116c msp: 0x20080000
	** Programming Started **
	auto erase enabled
	device id = 0x10016451
	flash size = 1024kbytes
	Single Bank 1024 kiB STM32F76x/77x found
	target halted due to breakpoint, current mode: Thread 
	xPSR: 0x61000000 pc: 0x20000046 msp: 0x20080000
	wrote 32768 bytes from file build/led.bin in 2.035821s (15.718 KiB/s)
	** Programming Finished **
	** Verify Started **
	target halted due to breakpoint, current mode: Thread 
	xPSR: 0x61000000 pc: 0x2000002e msp: 0x20080000
	verified 4612 bytes in 0.716718s (6.284 KiB/s)
	** Verified OK **
	> shutdown 
	shutdown command invoked
	Connection closed by foreign host.
	
	```
3. 按开发板上的 `RESET` 键后代码自动运行

> 以上3个步骤可以简写为一个步骤
> openocd -f /usr/share/openocd/scripts/interface/stlink-v2.cfg  \
> 	  -f /usr/share/openocd/scripts/target/stm32f7x.cfg \
> 	  -c "program build/led.bin 0x8000000 verify" \
> 	  -c "shutdown"

## openocd 烧写程序的命令格式

```shell
program <filename> [preverify] [verify] [reset] [exit] [offset]
```

## gdb 调试代码


由于 Ubuntu 20.04 不能通过`apt install` 的方式安装 `gdb-arm-none-eabi` , 具体安装方式可以参照[这个链接](https://askubuntu.com/questions/1243252/how-to-install-arm-none-eabi-gdb-on-ubuntu-20-04-lts-focal-fossa)

`arm-none-eabi-gdb`的使用方式可以参照[这篇文章](https://www.cnblogs.com/zongzi10010/p/10023535.html#gdb)中的GDB小节,
因此暂时不尝试。

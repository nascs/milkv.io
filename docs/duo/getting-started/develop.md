### How to switch function of pin
according to duo_pinmux.md, we need to list functions of a pin, for example

```bash
[root@cvitek]~# cvi_pinmux -r  IIC0_SCL
IIC0_SCL function:
[ ] JTAG_TDI
[ ] UART1_TX
[ ] UART2_TX
[v] XGPIOA_28
[ ] IIC0_SCL
[ ] WG0_D0
[ ] DBG_10

register: 0x300104c
value: 3


```

then we can switch function of a pin by devmem, for example, If we need to switch to the UART1_TX function of that pin, we can do the same as below

```bash
			//  devmem register  32 function_num
[root@cvitek]~# devmem 0x300104c 32 0x1

[root@cvitek]~# cvi_pinmux -r  IIC0_SCL 0x1
IIC0_SCL function:
[ ] JTAG_TDI     //function 0
[v] UART1_TX     //function 1
[ ] UART2_TX     //function 2
[ ] XGPIOA_28    //function 3
[ ] IIC0_SCL     //function 4
[ ] WG0_D0       //function 5
[ ] DBG_10       //function 6

register: 0x300104c
value: 1

```

### how to use i2c
* switch two pins of i2c to i2c function

```bash

[root@cvitek]~# cvi_pinmux -r SD1_CLK
SD1_CLK function:
[ ] PWR_SD1_CLK
[ ] SPI2_SCK
[ ] IIC3_SDA
[v] PWR_GPIO_23
[ ] CAM_HS0
[ ] EPHY_SPD_LED
[ ] PWR_SPINOR1_SCK
[ ] PWM_9

register: 0x30010a0
value: 3
[root@cvitek]~# devmem 0x30010a0 32 0x2

[root@cvitek]~# cvi_pinmux -r SD1_CMD
SD1_CMD function:
[ ] PWR_SD1_CMD
[ ] SPI2_SDO
[ ] IIC3_SCL
[v] PWR_GPIO_22
[ ] CAM_VS0
[ ] EPHY_LNK_LED
[ ] PWR_SPINOR1_MOSI
[ ] PWM_8

register: 0x300109c
value: 3

[root@cvitek]~# devmem 0x300109c 32 0x2

```


* connect i2c device with I2C3 (I2C3_SDA pin and I2C3_SCL pin)
* scan the i2c address by the following command

```bash
[root@cvitek]~# i2cdetect -r -y 3
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- -- 
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
40: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- 
60: -- -- -- -- -- -- -- -- 68 -- -- -- -- -- -- -- 
70: -- -- -- -- -- -- -- -- 

```

### how to use pwm 
The pwm driver is not loaded by default，we can see nothing under the folder /sys/class/pwm/ 

```bash
[root@cvitek]~# ls /sys/class/pwm
[root@cvitek]~#
```
so, we need to load the pwm driver if we need pwm

* load pwm driver

```bash
on host:
scp cvi_mmf_sdk/middleware/v2/ko/cv180x_pwm.ko root@192.168.2.196:~

ip of duo: 192.168.2.196 



on duo board:
[root@cvitek]~# insmod cv180x_pwm.ko
[root@cvitek]~# ls /sys/class/pwm/
pwmchip0  pwmchip12  pwmchip4  pwmchip8

```

* knowledge of pwm on duo 
Cv180x has four PWM IP's (pwmchip0/ pwmchip4/ pwmchip8/ pwmchip12), each IP control 4 signals, a total of 16 signals can be controlled (PWM0 is not controllable).

```
In Linux sysfs, the pwm0 to pwm3 device nodes are as follows:
/sys/class/pwm/pwmchip0/pwm0~3

In Linux sysfs, the pwm4 to pwm7 device nodes are listed as follows:
/sys/class/pwm/pwmchip4/pwm0~3

In Linux sysfs, the pwm8 to pwm11 device nodes are listed as follows:
/sys/class/pwm/pwmchip8/pwm0~3

In Linux sysfs, the pwm12 to pwm15 device nodes are listed as follows:
/sys/class/pwm/pwmchip12/pwm0~3
```

* switch pin function to pwm

```bash
[root@cvitek]~# cvi_pinmux -r SD1_D1
SD1_D1 function:
[ ] PWR_SD1_D1
[ ] IIC1_SDA
[ ] UART2_RX
[v] PWR_GPIO_20
[ ] CAM_MCLK1
[ ] UART3_RX
[ ] PWR_SPINOR1_WP_X
[ ] PWM_6

register: 0x3001094
value: 3
[root@cvitek]~# devmem 0x3001094 32 0x7

```
* use pwm by sysfs

```bash

[root@cvitek]~# echo 2 > /sys/class/pwm/pwmchip4/export 
[root@cvitek]~# echo 40000 > /sys/class/pwm/pwmchip4/pwm2/period 
[root@cvitek]~# echo 10000 > /sys/class/pwm/pwmchip4/pwm2/duty_cycle 
[root@cvitek]~# echo normal > /sys/class/pwm/pwmchip4/pwm2/polarity 
[root@cvitek]~# echo 1 > /sys/class/pwm/pwmchip4/pwm2/enable 

```


### how to using wiringX on duo

* get the tool-chain

```bash
on your host:

sudo apt-get install cmake build-essential git -y 
wget https://sophon-file.sophon.cn/sophon-prod-s3/drive/23/03/07/16/host-tools.tar.gz
tar xvf host-tools.tar.gz
```

* get the source code 

```bash
git clone -b duo https://github.com/nascs/wiringX.git
```

* modify the CMakeLists.txt

```bash
set (CMAKE_C_COMPILER "/path/to/riscv64-*-linux-gnu-gcc")
set (CMAKE_CXX_COMPILER "/path/to/riscv64-*-linux-gnu-++")
```

* build

```bash
mkdir build && mkdir install_dir
cd build 
cmake .. && make -j8 && make install DESTDIR=../install_dir 
cd ..
```

* cross compile

```bash
riscv64-unknown-linux-gnu-gcc ./examples/blink.c -lwiringx -I ./src/ -L install_dir/usr/local/lib/ -o blink --static 
```

* excute the program

```bash
on your host:
scp blink root@192.168.x.x:~

on board
./blink duo 1     #    ./blink duo gpio

```
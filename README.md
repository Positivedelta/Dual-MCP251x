# Dual-MCP251x Driver
Linux CAN driver with hardware filtering, configurable for up to two Microchip MCP251x devices  
e.g. for use with the Waveshare Dual CAN expansion board for the Raspberry Pi

Based on patch from:
https://support.criticallink.com/redmine/attachments/download/14913/0001-can-mcp251x-Add-ability-to-use-CAN-ID-hw-filter.patch

# Compiling the driver
First download the kernel headers:
```
$ sudo apt-get install raspberrypi-kernel-headers
```
Now download this repository
```
$ git clone https://github.com/Positivedelta/Dual-MCP251x.git
```
Make the kernel module:
```
$ cd Dual-MCP251x
$ make
```
If successful it should display something similar to below and create you a mcp251x.ko file
```
make -C /lib/modules/5.10.63-v8+/build M=/home/pi/development/Dual_MCP251x modules
make[1]: Entering directory '/usr/src/linux-headers-5.10.63-v8+'
  CC [M]  /home/pi/development/Dual_MCP251x/mcp251x.o
  MODPOST /home/pi/development/Dual_MCP251x/Module.symvers
  CC [M]  /home/pi/development/Dual_MCP251x/mcp251x.mod.o
  LD [M]  /home/pi/development/Dual_MCP251x/mcp251x.ko
make[1]: Leaving directory '/usr/src/linux-headers-5.10.63-v8+'
```

# Testing
First the configure and associate the SPI driver with the CAN hardware. On the Raspberry Pi this is
done by adding the following configuration to `/boot/config.txt`
```
dtparam=spi=on
dtoverlay=mcp2515-can1,oscillator=16000000,interrupt=25
dtoverlay=mcp2515-can0,oscillator=16000000,interrupt=23
```
Next, reboot the Raspberry Pi in order for the changes to take effect  
Finally to test the driver, remove the old one (if loaded) and insert your new module into the kernel using:
```
$ sudo rmmod mcp251x
$ sudo insmod mcp251x.ko
```
Then check your kernel messages:
```
$ dmesg | grep -i mcp251x
[ 1396.047462] mcp251x: loading out-of-tree module taints kernel.
[ 1582.644169] mcp251x spi0.0 can0: MCP2515 successfully initialized.
```
Parameters to specify the CAN filtering mode, masks and filters can be sent to the kernel using insmod  
This driver supports up to two MCP251x devices  
To configure a single device use (split to ease readability):
```
$ sudo insmod mcp251x.ko device=can0
                         mode1=1,1 mask1=0xfff,0xfff filt1=0x100,0x101,0x102,0x103,0x104,0x105
```
To configure two devices use (split to ease readability):
```
$ sudo insmod mcp251x.ko device=can0,can1
                         mode1=1,1 mask1=0xfff,0xfff filt1=0x100,0x101,0x102,0x103,0x104,0x105
                         mode2=1,1 mask2=0xfff,0xfff filt2=0x400,0x401,0x402
```
Notes:
* Each device supports two masks, one for each of its receive buffers
* Priority receive buffer1 supports up to 2 filters
* Receive buffer2 supports up to 4 filters
* Any omitted mask or filter values will be configured as `0x000`

To enable a device use (set for 1Mb/s):
```
sudo ip link set can0 up type can bitrate 1000000
```
Any configured hardware filters will be reported in the Kernel log and can be viewed using `dmesg`
```
[ 4685.423344] mcp251x spi0.1 can0: Configuring hardware filtering for the associated MCP2515
[ 4685.423373] mcp251x spi0.1 can0: - Modes: 1, 1
[ 4685.423394] mcp251x spi0.1 can0: - Masks: 0xfff, 0xfff
[ 4685.423416] mcp251x spi0.1 can0: - Filters: [0x100, 0x101], [0x102, 0x103, 0x104, 0x104]
```

Enabled CAN devices can now be seen in the network configuration, to display this use:
```
$ ifconfig
```
Note, the default transmit buffer size is relatively small, if necessary this can be updated using:
```
$ sudo ifconfig can0 txqueuelen 65536
```

Modinfo can be used to describe the hardware filtering configuration parameters:
```
$ modinfo mcp251x.ko
filename:       /home/pi/development/Dual_MCP251x/mcp251x.ko
license:        GPL v2
description:    Microchip 251x/25625 CAN driver
author:         Chris Elston <celston@katalix.com>, Christian Pellegrin <chripell@evolware.org>
srcversion:     4445A35007E8A511F24D760
alias:          of:N*T*Cmicrochip,mcp25625C*
alias:          of:N*T*Cmicrochip,mcp25625
alias:          of:N*T*Cmicrochip,mcp2515C*
alias:          of:N*T*Cmicrochip,mcp2515
alias:          of:N*T*Cmicrochip,mcp2510C*
alias:          of:N*T*Cmicrochip,mcp2510
alias:          spi:mcp25625
alias:          spi:mcp2515
alias:          spi:mcp2510
depends:        can-dev
name:           mcp251x
vermagic:       5.10.63-v8+ SMP preempt mod_unload modversions aarch64
parm:           enable_dma: Configure SPI to use DMA
                            0 = OFF (default value)
                            1 = ON (can use any non zero value) (int)
parm:           device:     Specify a maximum of TWO device names for hardware filter configuration
                            Notes, Filters will only be applied to matching devices
                                   When specifing two devices, add both sets of mode, mask and filter parameters (array of charp)
parm:           mode1:      Configures device1 hardware packet filtering for the receive buffers rxb0 and rxb1
                            Kernel level filtering can still be used as a secondary option, but may struggle to keep up in high traffic scenarios
                            Notes, Filters 0 and 1 apply to rxb0
                                   Filters 2, 3, 4 and 5 apply to rxb1
                            0 = Disable all hardware filtering (default value)
                            1 = Filter, but only accept std CAN IDs
                            2 = Filter, but only accept ext CAN IDs
                            3 = Filter and accept std or ext CAN IDs (array of int)
parm:           mode2:      Configures device2 hardware packet filtering for the receive buffers rxb0 and rxb1
                            See above device1 details for the mode configuration options (array of int)
parm:           mask1:      Mask used for the device1 receive buffer n, only applies to modes 1, 2 or 3
                            Use bits 10-0 for std IDs
                            Use bits 29-11 for ext IDs (array of int)
parm:           mask2:      Mask used for the device2 receive buffer n, only applies to modes 1, 2 or 3
                            See above device1 details for the mask configuration options (array of int)
parm:           filt1:      Filters used for the device1 receive buffers, only applies to modes 1, 2 or 3
                            Use bits 10-0 for std IDs
                            Use bits 29-11 for ext IDs (must set bit 30 for ext ID filtering)
                            Notes, Filters 0 and 1 correspond to mode1[0] and mask1[0]
                                   Filters 2, 3, 4 and 5 correspond to mode1[1] and mask1[1] (array of int)
parm:           filt2:      Filters used for the device2 receive buffers, only applies to modes 1, 2 or 3
                            See above device1 details for the mask configuration options (array of int)
```
Once the driver is loaded, you can check the current parameters via sysfs  
e.g. For the first configured device use:
```
$ cat /sys/module/mcp251x/parameters/device
can0
$ cat /sys/module/mcp251x/parameters/mode1
1,1
$ cat /sys/module/mcp251x/parameters/mask1
4095,4095
$ cat /sys/module/mcp251x/parameters/filt1
256,257,258,259,260,261

```
# Installation
To install the driver:
```
sudo cp mcp251x.ko /lib/modules/$(uname -r)/kernel/drivers/net/can/spi/
```
Parameters can be passed to the driver using the modprobe daemon  
Create a new file `/etc/modprobe.d/mcp251x.conf` and add the following (single device config shown):
```
options mcp251x device=can0 mode1=1,1 mask1=0xfff,0xfff filt1=0x100,0x101,0x102,0x103,0x105,0x105
```

# How is the connection being done

Hardware involved:
 - multimeter [avo410](https://uk.megger.com/digital-multimeter-avo410)
 - rs pro 460-9881
 - usb-rs-232 converter - [VOTEK FTDI Chipset Plugable USB to RS232](https://www.amazon.co.uk/Chipset-Plugable-adapter-support-Windows/dp/B01CCNR7W6/ref=cm_sw_em_r_dp_da_3Qr-ybV39D8KR_tt)

# When we connected everything

The only indication things are working is a continuously red LED on the rs-232 end of the VOTEK cable. 

It is not clear what the red LED means.

# Can we find a device that matches the VOTEK cable?

We connected the setup to a usb port and then run:

	ls /dev cableConnected

We then unplugged the cable from the usb port and run:

	ls /dev cableDisconnected

Diffing both files gave us the following

	# diff cableConnected cableDisconnected
	152d151
	< serial
	265d263
	< ttyUSB0

`ttyUSB0` is a device.

`serial` was a directory. To understand it, we did

	alexmsmartins@Altair:~$ tree /dev/serial/
	/dev/serial/
	├── by-id
	│   └── usb-FTDI_FT232R_USB_UART_A600ECO3-if00-port0 -> ../../ttyUSB0
	└── by-path
	    └── pci-0000:00:14.0-usb-0:2:1.0-port0 -> ../../ttyUSB0

We also tried to plug ONLY the VOTEK converter in different ports to see which devices/symlinks would change.


	alexmsmartins@Altair:~$ tree /dev/serial/
	/dev/serial/
	├── by-id
	│   └── usb-FTDI_FT232R_USB_UART_A600ECO3-if00-port0 -> ../../ttyUSB0
	└── by-path
	    └── pci-0000:00:14.0-usb-0:3:1.0-port0 -> ../../ttyUSB0

	2 directories, 2 files
	alexmsmartins@Altair:~$ tree /dev/serial/
	/dev/serial/
	├── by-id
	│   └── usb-FTDI_FT232R_USB_UART_A600ECO3-if00-port0 -> ../../ttyUSB0
	└── by-path
	    └── pci-0000:00:14.0-usb-0:1:1.0-port0 -> ../../ttyUSB0

Only the /dev/serial/by-path changes in the usb-0:*1* part of the path according to the specific hardware usb port we plugged it into.

# Reading data from the Multimeter

By plugging the multimeter to the cables as explained above we started trying to get data from it.

We first tried to:
 - set the multimter to measure resistance
 - connect the tips to resistors 
 - cat the device in linux

Nothing appeared in the console but, after pessing the RS232 button, garbage characters kept coming at an inconsistent rate. Tow yellow LEDs also started blinking in the VOTEK converter with a fast and variable frequency. 


	alexmsmartins@Altair:~$ sudo cat /dev/serial/by-id/usb-FTDI_FT232R_USB_UART_A600ECO3-if00-port0 
	ĥE�
		      �ͬ
		       ���E���͌bJ�E���bz���E���H�bJͥ�E���͌bJ�$�E��ͬbJL��E���ͬbJ���E�̅ͬbJ���E��$ͬbJ��E���bJ���E��ͬbJ��E���ͬ���E��
	...

According to Monica, this is sometimes fixed by setting the serial port's baud rate.

# How to fix the baud rate in linux

First things first, we needed to avoid using sudo :

	alexmsmartins@Altair ~> ls -la /dev/ttyUSB0 
	crw-rw---- 1 root dialout 188, 0 Apr 20 23:06 /dev/ttyUSB0
	alexmsmartins@Altair ~> 
	sudo chmod o+rw /dev/ttyUSB0 
	alexmsmartins@Altair ~> ls -la /dev/ttyUSB0 
	crw-rw-rw- 1 root dialout 188, 0 Apr 20 23:06 /dev/ttyUSB0

We also choose to use pyserial for now in hopes it makes our lifes easier.

Check https://pythonhosted.org/pyserial/shortintro.html 

We managed to get hexadecimal representations of the received bytes with the following code

	In [1]: import serial
	In [9]: ser = serial.Serial('/dev/ttyUSB0')  # open serial port
	In [5]: ser.write(b'hello')
	Out[5]: 5
	In [17]: ser.read(100)
	Out[17]: b'\x81\x08\xa4\xa4\x06E\xab\x8c\xad\xcd\xacB\x04\x86\xa5\x86D\xab\xcc\xac\xcd\x8cb\x00\xc4\xa6\x06E\xab\x8c\xc6\xcd\xacb\xa4\x8c\xac\x8eE\x8b\x8d\x8c\xcd\x08bj\x0c,\x8e\x10\xab\x8d\x8c\xcd\x8cb\xa5cR\x02E\xab\xcc\xc6\xcd\xac\x02\x08\xcc\xad\x06E\xab\x8c.\xcd\xacb\t\xac\xac\x86D\xab\x8c\x85\xcd\xacb\x84\x8d\xa4\x86E\x8b\xa4\x8c\xcd\xacb'



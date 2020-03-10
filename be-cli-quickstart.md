### be-cli工具使用说明

#### 列出当前的串口列表：`be devices`

示例：
```bash
$ be devices
2020-3-10 12:04:55:596 device-agent /device-agent/scan {"cli":true}
2020-3-10 12:04:57:618 [0] serialport: /dev/tty.usbmodem14103
```

#### 制作app.bin包：`be packApp <appDir> <outDir>`

`<appDir>`为需要打包的app目录，`<outDir>`为app.bin的存放目录。

示例：
```bash
$ be packApp ./app ./
2020-3-10 12:06:42:963 device-agent /device-agent/packApp {"appDir":"./app","outDir":"./","cli":true}
2020-3-10 12:06:42:967 ============verifyJSEPACK==========
2020-3-10 12:06:42:967 buf 长度 = 574
2020-3-10 12:06:42:967 file_count = 2 pack_version=2 pack_size=574
2020-3-10 12:06:42:969 be-cli packApp success
```

#### 烧录app.bin到设备中：`be flashAppBin ./app.bin developerkit`

示例：
```bash
$ be flashAppBin ./app.bin developerkit
2020-3-10 12:03:11:937 device-agent /device-agent/flashAppBin {"appBinPath":"./app.bin","platform":"developerkit","cli":true}
2020-3-10 12:03:11:940 device-agent /device-agent/unpack {"appBinPath":"./app.bin","outDir":"/var/folders/c2/6f6jgl713f7c43_yxj1krzf80000gp/T/1583812991939"}
2020-3-10 12:03:11:943 unpackApp success
2020-3-10 12:03:11:944 device-agent /device-agent/mkspiffs {"appDir":"/var/folders/c2/6f6jgl713f7c43_yxj1krzf80000gp/T/1583812991939","outFile":"/var/folders/c2/6f6jgl713f7c43_yxj1krzf80000gp/T/1583812991943.spiffs.bin","platform":"developerkit"}
2020-3-10 12:03:11:944 device-agent mkspiffs argv ["-c","/var/folders/c2/6f6jgl713f7c43_yxj1krzf80000gp/T/1583812991939","-b","16384","-p","2048","-s","262144","/var/folders/c2/6f6jgl713f7c43_yxj1krzf80000gp/T/1583812991943.spiffs.bin"]
2020-3-10 12:03:11:954 mkspiffs /board.json

2020-3-10 12:03:11:955 mkspiffs /index.js

2020-3-10 12:03:11:959 mkspiffs success
2020-3-10 12:03:11:960 device-agent /device-agent/flashBin {"binPath":"/var/folders/c2/6f6jgl713f7c43_yxj1krzf80000gp/T/1583812991943.spiffs.bin","platform":"developerkit","address":"0x080C0000"}
2020-3-10 12:03:11:960 st-flash ["--reset","write","/var/folders/c2/6f6jgl713f7c43_yxj1krzf80000gp/T/1583812991943.spiffs.bin","0x080C0000"]
2020-3-10 12:03:12:043 flash st-flash 1.5.0
Flash page at addr: 0x080c0000 erased
......
Flash page at addr: 0x080ff800 erased
2020-3-10 12:03:23:642 flash
size: 32768
size: 32768
size: 32768
size: 32768
size: 32768
size: 32768
size: 32768
size: 30832

2020-3-10 12:03:23:643 flashBin success
2020-3-10 12:03:23:644 device-agent flash success
```

#### 解压app.bin包：`be unPackApp <appBinPath> <outDir>`

示例：
```bash
$ be unpackApp app.bin ./xxx
2020-3-10 12:10:58:842 device-agent /device-agent/unpack {"appBinPath":"app.bin","outDir":"./xxx","cli":true}
2020-3-10 12:10:58:847 be-cli unpackApp success
$ ll xxx
total 16
-rw-r--r--  1 xxw  staff   241B  3 10 12:10 board.json
-rw-r--r--  1 xxw  staff   259B  3 10 12:10 index.js
```
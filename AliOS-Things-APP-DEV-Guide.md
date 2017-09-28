# 目录
  * [1 概述](#1概述)
  * [2 硬件环境准备](#2硬件环境准备)
  * [3 开发环境搭建](#3开发环境搭建)
  * [4 应用开发步骤](#4应用开发步骤)
  * [5 第一个AliOS应用](#5第一个AliOS应用)
  * [6 开发组件介绍](#6开发组件介绍)
  * [7 总结](#7总结)

---

# 1概述
本文将描述如何基于AliOS Things源码进行应用开发，涉及的内容包括：软硬件环境搭建、如何创建第一个应用程序、AliOS Things重要开发组件的使用等。

# 2硬件环境准备
AliOS Things可以运行在各种硬件平台上。开发应用的硬件环境包括开发板、串口、调试器、烧录器等，详细的硬件环境搭建请参考[AliOS Things Development Environment Setup](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-Environment-Setup)

# 3开发环境搭建
AliOS Things的开发支持IDE（AliOS Things Studio）和命令行工具，AliOS Things开发环境的搭建请参照：[AliOS Things Development Environment Setup](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-Environment-Setup)

# 4应用开发步骤
基于AliOS Things可以很方便地进行应用开发。基于AliOS Things创建应用，在IDE环境下可以通过导入应用模版的方式，在非IDE环境下可以手动创建各种工程目录和文件。
## 4.1 在非IDE环境中进行应用开发
非IDE环境中的应用开发步骤主要包括工程目录的创建、工程Makefile编写、源码编写、工程编译、程序烧录、调试等步骤。
### 4.1.1 创建工程目录
AliOS Things的应用工程一般放在“example”目录下，用户也可以根据需要在其他目录下创建应用工程的目录。
### 4.1.2 添加Makefile
Makefile用于指定应用的名称、使用到的源文件、依赖的组件、全局符号等。下面是sample.mk样例文件的内容：
```
NAME := helloworld  ## 指定应用名称
$(NAME)_SOURCES := helloworld.c  ## 指定使用的源文件
$(NAME)_COMPONENTS += cli  ## 指定依赖的组件，本例使用cli组件
$(NAME)_DEFINES += LOCAL_MACRO ## 定义局部符号
GLOBAL_DEFINES += GLOBAL_MACRO ## 定义全局符号
```
### 4.1.3 添加源码
所有的源码文件放置在应用工程目录下，开发者可以根据自行组织源码文件/目录。AliOS Things的应用程序入口为：
`int application_start(int argc, char *argv[]);`

所有的应用程序都必须包含`application_start`入口函数，应用程序的逻辑从该入口函数开始。
### 4.1.4 编译、烧录和调试
应用的编译、烧录和调试可以参考：[Linux开发环境](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-Environment-Setup#3-linux-环境配置)。
## 4.2 在AliOS Things Studio IDE中进行应用开发
AliOS Things提供了AliOS Things Studio集成开发环境，基于AliOS Things Studio进行应用开发非常方便、快捷。AliOS Things提供了可供导入的应用模版，用户可以基于导入的模版进行应用开发。在AliOS Things Sutdio IDE中，也可以很方便地进行编译、烧录、调试等
### 4.2.1 创建应用项目
关于如何在AliOS Things Studio中创建应用，请参考[使用AliOS Things Studio创建应用](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-Studio#22-创建-app-项目)。创建完项目后，用户可以在AliOS Things Studio中添加、编辑应用代码。
### 4.2.2 编译、烧录和调试
AliOS Things Studio IDE环境下的编译、烧录和调试步骤，可以参照：[IDE开发环境](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-Environment-Setup#2-window-环境配置)。

# 5第一个AliOS Things应用
本节以helloworld工程为例来说明如何创建一个AliOS Things应用（基于非IDE环境）。
## 5.1 创建工程目录
在“example”目录下添加helloworld工程目录。

## 5.2 创建Makefile
在hellworld工程目录下，创建helloworld.mk文件，并添加Makefile内容：
```
NAME := helloworld
$(NAME)_SOURCES := helloworld.c
```
## 5.3 创建源文件
在hellworld工程目录下，创建helloworld.c文件，并添加以下源代码：
```c
#include <aos/aos.h>

static void app_delayed_action(void *arg)
{
    printf("%s:%d %s\r\n", __func__, __LINE__, aos_task_name());
    aos_post_delayed_action(5000, app_delayed_action, NULL);
}

int application_start(int argc, char *argv[])
{
    aos_post_delayed_action(1000, app_delayed_action, NULL);
    aos_loop_run();
}
```

## 5.4 编译、烧录和运行
请按照前述章节对helloworld应用进行编译和烧录。烧录完成后启动开发板，应用程序会被自动执行。helloworld应用启动后串口打印如下：

![](https://img.alicdn.com/tfs/TB11fSrdwMPMeJjy1XdXXasrXXa-231-161.png)

# 6开发组件介绍
AliOS Things提供了丰富的组件来支持IoT应用的开发。
## 6.1 yloop
yloop是一个异步事件框架，主要负责管理系统各类事件的分发处理，及各类微任务（action）的调度。基于yloop，开发者可以避免多线程编程引入的复杂度和资源占用。yloop支持监听本地事件和网络事件，支持延时调用，支持workqueue处理耗时事件。AliOS Things系统起来后有一个main yloop，也支持任务创建属于自己的yloop。yloop提供了注册，发送事件的接口。开发者可以用这些接口编写基于事件监听机制的程序，以及和系统其他组件的消息通信。[yloop接口介绍](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-API-YLOOP-Guide)
## 6.2 Kernel
kernel是AliOS Things的核心组件之一，其基础是代号为Rhino的实时操作系统。AliOS Things kernel实现了多任务机制，多个任务之间的调度，任务之间的同步、通讯、互斥、事件，内存分配，trace功能，多核等等的机制。开发者可以利用kernel提供的api来实现一个RTOS所具备的能力。开发者可以利用现有已移植的CPU体系架构来达到快速的移植能力。[Kernel接口介绍](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-API-KERNEL-Guide)
## 6.3 Alink
Alink组件提供开放丰富安全可靠的云服务，可以用于Alink上云连接服务，如配网、数据上报等。借助Alink组件，用户可以很方便的实现实现用户与设备、设备与设备、设备与用户之间的互联互动。关于Alink组件详细的介绍以及接口的定义，请参考：[Alink接口介绍](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-API-ALINK-Guide)
## 6.4 硬件抽象层（HAL）
HAL是AliOS Things的核心组件之一，主要目的是为了屏蔽底下不同的芯片平台的差异，从而使上面的应用软件不会因为不同的芯片而改变。目前ALiOS Things定义了丰富的HAL抽象层，芯片公司或者用户只要对接相应的HAL接口即能满足控制芯片的控制器，从而达到控制硬件外设的目的。HAL包含的功能有adc，flash， gpio，i2c，pwm，rng，rtc，sd，spi，timer，uart，wdg。开发者可以利用HAL的API来快速达到控制硬件外设的能力。由于目前的HAL层是非常标准的API，开发者可以参考现有移植的HAL层的开发，来达到快速移植的能力。[HAL接口介绍](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-HAL-Porting-Guide)
## 6.5 KV
KV组件是基于键(key)-值(value)数据类型的小型轻量级持久化存储组件，主要为基于Nor-Flash的小型嵌入式设备提供通用的key-value持久化存储接口。KV组件支持写平衡（磨损平衡）与掉电保护；而且，KV组件的资源占用较低，RAM占用峰值固定，只和最大键值对长度相关。KV组件封装了简洁的增加/修改、删除及查询的接口方法，开发者可以通过这些接口方法存储一些产品参数、系统配置等信息。KV组件的移植请参照：[KV移植](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-HAL-Porting-Guide#2kv组件移植开发注意事项)。[KV接口介绍](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-API-KV-Guide)
## 6.6 VFS
VFS是AliOS Things核心组件之一，用户可以使用VFS组件通过统一的接口操作不同的真实文件系统和设备。VFS组件是对真实文件系统操作与设备操作的一层抽象层，VFS组件封装了一套对文件对象与设备对象执行操作的统一接口，具有良好的跨平台性。开发者可以通过VFS组件封装的统一接口（aos_open、aos_read，etc.）来操作真实文件系统与设备。而且利用VFS组件的统一接口来进行应用开发，可以让应用具有更好的跨平台性，不再依赖设备HAL层接口定义和各个真实文件系统的接口定义。[VFS接口介绍](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-API-VFS-Guide)
## 6.7 uMesh
uMesh是AliOS Things核心组件之一，模组之间通过uMesh能够形成自组织网络。uMesh实现了mesh链路管理、mesh路由、6LoWPAN、AES-128数据加解密等。它能够支持mesh原始数据包、IPv4或IPv6多种数据传输方式。开发者可以使用熟悉的socket编程，利用uMesh提供的自组织网络实现智能设备的开发和互连，能够使用在智能照明，智能抄表，智能家居等场景。开发者也可以通过实现uMesh提供的mesh HAL层接口，将uMesh移植到不同的通信介质，如WiFi，802.15.4, BLE等。
## 6.8 CLI
AliOS Things应用开发中可以支持命令行，并且可以添加用户自定义命令。当需要注册或撤销自定义命令时，可以借助“CLI”组件。[CLI接口介绍](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-API-CLI-Guide)

下图展示了一个实际示例应用中的命令列表。

![](https://img.alicdn.com/tfs/TB1ETiGdwMPMeJjy1XcXXXpppXa-447-367.png)

# 7总结
本文描述了基于AliOS Things的应用模型，介绍了软硬件开发环境的搭建、应用开发的基本步骤。以helloworld为例，展示了如何基于AliOS Things进行应用开发。本文最后，还介绍了AliOS Things提供的丰富组件和接口，以及如何利用这个组件进行应用开发。
想了解AliOS Things更详细的信息，请访问[AliOS Things Github主页](https://github.com/alibaba/AliOS-Things)。
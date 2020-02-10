**Table Of Content**

1. [概述](#1-概述)
* 1.1 [SAL简介](#11-sal简介)
* 1.2 [SAL架构与组成](#11-sal简介)
* 1.3 [SAL驱动概述](#13-sal驱动概述)
2. [准备工作](#2-准备工作)
3. [驱动框架自动生成](#3-驱动框架自动生成)
4. [驱动对接](#4-驱动对接)
* 4.1 [串口和AT命令特征配置](#41-串口和at命令特征配置)
* 4.2 [添加at设备](#42-添加at设备)
* 4.3 [HAL接口实现](#43-hal对接)
* 4.4 [atparser使用指南](#44-atparser使用指南)
* 4.5 [特别提醒](#45-hal对接特别提醒)
5. [驱动集成](#5-驱动集成)
* 5.1 [menuconfig集成](#51-menuconfig集成)
* 5.2 [C源码集成](#52-c源码集成)
* 5.3 [Makefile集成](#53-makefile集成)
* 5.4 [板级集成](#54-板级集成)
6. [驱动使用与验证](#6-驱动使用与验证)
* 6.1 [硬件准备](#61-硬件准备)
* 6.2 [测试app开发/添加](#62-测试app开发添加)
* 6.3 [在app中完成模组设备添加](#63-在app中完成模组设备的添加)
* 6.4 [使能board外挂模组的串口](#64-使能board外挂模组的串口)
* 6.5 [配置、编译和运行](#65-配置编译和运行)
7. [Keil、IAR工程实践](#7-keil和iar工程实践)
8. [参考文档](#8-参考文档)
9. [总结](#9-总结)

# 1 概述
本文适用于使用MCU外挂AT模组方式接入AliOS Things和阿里云物联网平台的用户。本文的目标是为模组对接用户提供完整、详细的指导说明，使用户在只参考本文的情况下，能快速完成广域网模组的接入。

本文将介绍AliOS Things的AT模组SAL驱动架构，详细讲述进行AT模组SAL驱动对接的步骤和流程，包括源码获取、工具安装、SAL脚手架的使用、AT模组对接和集成、测试与验证等。本文中所列举的示例，均基于stm32f103rb-nucleo（MCU）+ m5310a（NBIoT模组）的平台组合。

## 1.1 SAL简介
SAL，是Socket Adapter Layer的简称。AliOS Things中SAL套件基于外挂AT模组的方式，提供标准Socket接口服务。SAL套件提供AT命令到Socket标准接口的转换。借助SAL套件，用户不用感知底层通信方式和介质（如WiFi、2G、4G等模组），可以使用标准Socket接口进行应用开发，使上层应用具有更好的可移植性。

## 1.2 SAL架构与组成
下图是AliOS Things中SAL套件的架构示意图。

![SAL组件结构](https://img.alicdn.com/tfs/TB1.VNJvpT7gK0jSZFpXXaTkpXa-1999-1125.png)
其中，组件包括：
* SAL Core：由AliOS Things提供，SAL核心组件。主要包括Socket连接管理、数据缓存、协议转换等功能，对上提供标准Socket接口服务，对下提供统一的HAL接口规范（可以对接到不同厂商的AT模组）。
* SAL Driver：驱动，部分（市场TOP系列）由AliOS Things提供，其他由用户自己提供。SAL驱动模块基于具体型号的通信模组提供的AT命令，实现SAL规范的HAL接口功能。

API包括：
* Socket API：这一层接口提供标准的socket服务，请参考socket接口手册。SAL提供的socket服务是标准socket服务的子集，其中TCP client相关的接口一般都支持，但TCP server、UDP client等相关的接口支持与否，视具体模组而定。
* SAL HAL API：这一层接口定义SAL核心模块与不同厂商模组驱动之间的统一界面。这一层HAL的对接是模组驱动接入中的主要工作。
* AT Interface：这一层用于处理AT交互，如AT命令收发、事件回调。对应地，AliOS Things提供了atparser组件供用户选择使用，后续章节会有详细介绍。

## 1.3 SAL驱动概述
模组的SAL驱动，提供SAL规范的HAL接口的具体实现。SAL HAL提供统一规范的接口，模组驱动提供不同模组的具体实现，从而实现SAL对各厂商不同模组的支持。

SAL驱动主要实现类似Socket数据发送、数据接收、域名/IP转换这样的功能。AliOS Things中提供了不同类型模组（WiFi、2G、4G等）的参考实现，用户可以参考这些示例进行自己模组的对接。

SAL驱动的对接和集成的细节，在后续章节中介绍，可以分为5个步骤。
![对接步骤](https://img.alicdn.com/tfs/TB1cxRHvqL7gK0jSZFBXXXZZpXa-1812-760.png)

# 2 准备工作
本文的介绍和示例，都基于AliOS Things 3.0版本代码和编译工具。用户在对接和集成模组驱动之前，需要准备好AliOS Things的代码和配套工具。如果用户需要运行本文介绍的示例驱动，还需要准备好stm32f103rb-nucleo（MCU）+ m5310a（NBIoT模组）硬件组合、固件及烧录工具等。
* 从Github获取AliOS Things源码，适用版本为dev_sal_driver，该版本为基于rel_3.0.0的分支。
* 安装aos-cube工具，请参考[这里](https://github.com/alibaba/AliOS-Things/wiki/AliOS-Things-Environment-Setup)。

以下章节讲详细介绍如何对接和集成一个新的模组驱动。

# 3 驱动框架自动生成
为简化SAL驱动的开发，AliOS Things配套的aos-cube工具，提供了自动生成模组驱动目录和框架的功能，使用户可以快速进入驱动的实现环节。
参考以下命令，快速生成SAL驱动目录和框架：

```shell
$ cd <aos源码目录>
$ aos create saldriver <module_name> -t <module_type>
[Info] Drivers Initialized at: /<path_to_aos_src>/drivers/sal/<module_type>/<module_name>
```

 其中，`module_name`是模组的名称，如mk3060（庆科WiFi模组）、m5310a（中移物联网NBIoT模组）等；`module_type`为模组类型（如wifi、gprs、lte）。

目前支持的SAL外接AT模组的类型包括：

- wifi（参考drivers/sal/wifi/bk7231/）
- gprs（参考drivers/sal/gprs/sim800/）
- lte（参考drivers/sal/lte/m02h）

**备注**：AliOS Things中即将推出NBIoT类型模组的支持，敬请期待。目前AliOS Things 3.0配套的工具和模版还不支持NBIoT类型模组，用户如有需要，可以临时选用其他类型进行对接（模组类型目前只起归类划分的作用，不影响驱动的具体实现）。

以m5310a模组为例：

```shell
$ cd ～/src/AliOS-Things
$ aos create saldriver m5310a -t nbiot
[Info] Drivers Initialized at: ～/src/AliOS-Things/drivers/sal/nbiot/m5310a
$ tree drivers/sal/nbiot/m5310a/
drivers/sal/nbiot/m5310a/
├── aos.mk
├── atcmd_config_module.h
├── Config.in
├── m5310a.c
└── README.md

0 directories, 5 files
```

**备注**：由于脚手架工具暂时不支持nbiot类型，建议用户使用lte类型自动生成框架，然后通过目录/文件重命名产生nbiot/m5310a驱动目录及文件。具体操作步骤如下：

```shell
$ aos create saldriver m5310a -t gprs
$ mkdir -p drivers/sal/nbiot/
$ mv drivers/sal/gprs/m5310a drivers/sal/nbiot/
$ sed -i '/.*m5310a.*/d' drivers/sal/gprs/Config.in
$ cp drivers/sal/gprs/Config.in drivers/sal/nbiot/
$ sed -i 's/gprs/nbiot/g' drivers/sal/nbiot/Config.in
$ sed -i 's/GPRS/NBIOT/g' drivers/sal/nbiot/Config.in
$ sed -i 's/SIM800/M5310A/g' drivers/sal/nbiot/Config.in
$ sed -i 's/sim800/m5310a/g' drivers/sal/nbiot/Config.in
```

# 4 驱动对接
本节介绍如何完成模组驱动的对接和开发。

在驱动目录和框架生成之后，用户需要对其中某些文件进行填充、修改和接口实现，主要是基于自身模组AT提供的功能和操作，实现SAL HAL规范的接口。
![驱动对接步骤](https://img.alicdn.com/tfs/TB1OfYguKbviK0jSZFNXXaApXXa-1812-364.png)

本章节中红色字体部分，是需要重点关注的信息。

## 4.1 串口和AT命令特征配置
如果使用了at parser组件来与模组进行AT交互，需要把串口的配置、AT指令配置等参数传给atparser组件。串口的配置，包括串口波特率、校验方式等参数的配置；AT命令的配置，包括命令发送结束符、命令回复特征等配置。这些配置项，需要在驱动对接时，通过`at_add_dev()`函数调用传递给atparser模块，atparser组件将他们用于AT命令收发控制的特征提取。

这些配置项的定义，一般放在驱动目录下，如`drivers/sal/nbiot/m5310a/atcmd_config_module.h`文件中，其中宏名称建议（但不是必须）采用以下表格中所列的，宏定义的值由用户根据自身模组的情况进行修改。

建议宏名称如下：

| 宏名称 | 含义 | 说明 |
| --- | --- | --- |
| AT_RECV_PREFIX | AT接收结束符。 | 一般是回车加换行。 |
| AT_RECV_SUCCESS_POSTFIX | AT命令成功的回复。 | 一般是“OK\r\n”。 |
| AT_RECV_FAIL_POSTFIX | AT命令失败的回复。 | 一般是“ERROR\r\n”。 |
| AT_SEND_DELIMITER | AT命令+数据发送时二者之间的分隔符，根据模组需要，可选。 | 例如“\r”。 |
| AT_UART_BAUDRATE | 串口波特率 | 如9600，115200等。 |
| AT_UART_DATA_WIDTH | 串口数据位宽 | 该宏的取值需要用AliOS Things中aos/hal/uart.h中定义的值，如DATA_WIDTH_8BIT。 |
| AT_UART_PARITY | 串口校验 | 同上。 |
| AT_UART_STOP_BITS | 串口停止位 | 同上。 |
| AT_UART_FLOW_CONTROL | 串口数据流控 | 同上。 |
| AT_UART_MODE | 串口工作模式 | 同上。 |


以m5310a模组为例的完整示例：
```c
/*
 * Copyright (C) 2015-2017 Alibaba Group Holding Limited
 */

#ifndef _ATCMD_CONFIG_MODULE
#define _ATCMD_CONFIG_MODULE

#include "aos/hal/uart.h"

/* prefix postfix delimiter */
#define AT_RECV_PREFIX "\r\n"
#define AT_RECV_SUCCESS_POSTFIX "OK\r\n"
#define AT_RECV_FAIL_POSTFIX "ERROR\r\n"
#define AT_SEND_DELIMITER "\r"

/* uart config */
#define AT_UART_BAUDRATE 9600
#define AT_UART_DATA_WIDTH DATA_WIDTH_8BIT
#define AT_UART_PARITY NO_PARITY
#define AT_UART_STOP_BITS STOP_BITS_1
#define AT_UART_FLOW_CONTROL FLOW_CONTROL_DISABLED
#define AT_UART_MODE MODE_TX_RX

typedef struct {
   uart_dev_t            uart_dev;
} sal_device_config_t;

#endif
```

## 4.2 添加AT设备
SAL设备添加处理，该函数添加该模组为SAL处理设备。在该函数中，通过`at_init`和`at_add_dev`接口向系统注册该AT模组设备。如果调用者通过`data`参数传入了SAL设备配置参数，则应该使用传入的参数进行设备添加操作，否则采用默认的参数（由对接层自己定义）进行设备添加。

以m5310a模组为例，下面是m5310a模组的m5310a_sal_add_dev接口的实现片段：

```c
static int m5310a_sal_add_dev(void* data)
{
    at_config_t at_config = { 0 };

    /*
     * 如果用户传入的配置信息，则使用用户指定的参数，
     * 否则不使用默认的（如果没有默认的，则提示错误信息）。
     */
    if(data != NULL) {
        sal_device_config_t* config = (sal_device_config_t *)data;
        uart_dev.port  = config->uart_dev.port;
        uart_dev.config.baud_rate    = config->uart_dev.config.baud_rate;
        uart_dev.config.data_width   = config->uart_dev.config.data_width;
        uart_dev.config.parity       = config->uart_dev.config.parity;
        uart_dev.config.stop_bits    = config->uart_dev.config.stop_bits;
        uart_dev.config.flow_control = config->uart_dev.config.flow_control;
        uart_dev.config.mode         = config->uart_dev.config.mode;
    } else {
        /* uart_dev should be maintained in whole life cycle */
        uart_dev.port                = AT_UART_PORT;
        uart_dev.config.baud_rate    = AT_UART_BAUDRATE;
        uart_dev.config.data_width   = AT_UART_DATA_WIDTH;
        uart_dev.config.parity       = AT_UART_PARITY;
        uart_dev.config.stop_bits    = AT_UART_STOP_BITS;
        uart_dev.config.flow_control = AT_UART_FLOW_CONTROL;
        uart_dev.config.mode         = AT_UART_MODE;
    }

    /* 调用at_init初始化atparser模块。 */
    at_init();

    /* configure and add one uart dev */
    at_config.type                             = AT_DEV_UART;
    at_config.port                             = uart_dev.port;
    at_config.dev_cfg                          = &uart_dev;
    at_config.send_delimiter                   = AT_SEND_DELIMITER;
    at_config.reply_cfg.reply_prefix           = AT_RECV_PREFIX;
    at_config.reply_cfg.reply_success_postfix  = AT_RECV_SUCCESS_POSTFIX;
    at_config.reply_cfg.reply_fail_postfix     = AT_RECV_FAIL_POSTFIX;

    /* 调用at_add_dev添加AT设备。 */
    if ((at_dev_fd = at_add_dev(&at_config)) < 0) {
        M5310A_LOGE(TAG, "AT parser device add failed!\n");
        return -1;
    }

    return 0;
}
```

## 4.3 HAL对接
这部分工作主要是对接实现SAL组件定义的几个HAL接口，这些接口将被SAL调用。模组对接时，需要提供这些HAL接口的实现，然后集成到SAL框架中。本章节将介绍HAL接口定义，以具体示例展示如何进行HAL对接。

根据前述章节的介绍，通过aos_cube工具生成驱动框架后，HAL接口的对接工作主要在drivers/sal/<module_type>/<module_name>/<module_name>.c文件中，以下是工具生成的驱动HAL接口相关的代码：

```c
/* Don't modify */
static sal_op_t sal_op = {
    .next = NULL,
    .version = "1.0.0",
    .name = "m5310a",
    .add_dev = m5310a_sal_add_dev,
    .init = HAL_SAL_Init,
    .start = HAL_SAL_Start,
    .send_data = HAL_SAL_Send,
    .domain_to_ip = HAL_SAL_DomainToIp,
    .finish = HAL_SAL_Close,
    .deinit = HAL_SAL_Deinit,
    .register_netconn_data_input_cb = HAL_SAL_RegisterNetconnDataInputCb,
};
```

HAL对接，即实现上述结构体中定义的几个接口函数，包括`HAL_SAL_Init、HAL_SAL_Start`、`HAL_SAL_Send`、`HAL_SAL_DomainToIp`、`HAL_SAL_Close`、`HAL_SAL_Deinit`、`HAL_SAL_RegisterNetconnDataInputCb`。

以下是每个HAL接口需要实现的功能和要求介绍（同时附带m5310a的实例展示）：

- **HAL_SAL_Init**
  - 申请操作AT模组所需的相关系统资源，如锁、信号量等；取决于自身驱动实现的需求。
  - 初始化AT模组，如关闭回显、设置波特率、关闭流控、等待网络连接等。需要根据各模组的特点和要求采取相应的初始化逻辑，目标是通过完成该函数调用后，AT模组可以正常使用。
  - AT事件（如数据到达、WiFi连接/断开等事件）处理函数注册（通过at_register_callback接口），最少需要注册Socket数据接受处理函数，后续需要在有数据到达时，通过`HAL_SAL_RegisterNetconnDataInputCb`接口注册的回调函数，将数据送至上层进行处理。此外，在AT模组获取到IP地址后，对接层需要通过`aos_post_event`接口向上层通知`CODE_WIFI_ON_GOT_IP`事件。
  - 以m5310a模组为例，下面是m5310a模组的HAL_SAL_Init接口的实现和注释说明：

```c
static int HAL_SAL_Init(void)
{
    // ...

    /* 锁、信号量等资源申请 */
    if (0 != aos_mutex_new(&g_link_mutex)) {
        M5310A_LOGE(TAG, "Creating link mutex failed (%s %d).", __func__, __LINE__);
        return -1;
    }

    // ...
    
    /*
     * g_link结构体初始化，用于存放socket连接相关的信息，
     * 如对应的socket fd、at fd，socket类型，远端信息等。
     */
    memset(g_link, 0, sizeof(g_link));
    for (link_id = 0; link_id < LINK_ID_MAX; link_id++) {
        g_link[link_id].fd = -1;
        g_link[link_id].at_fd = -1;
        g_link[link_id].socktype = M5310A_SOCK_UNKNOWN;
    }

    /* 注册AT事件回调处理函数 */
    at_register_callback(at_dev_fd, M5310A_SOCKET_DATA_EVENT_PREFIX, NULL,
                         NULL, 0, m5310a_socket_data_event_handler, NULL);
 
    // ...
    
    /* 以下是模组的初始化操作，包括等待模组ready、功能设置、网络连接、信号强度检测。 */
    ret = m5310a_net_attach(1);
    if (ret) {
        M5310A_LOGE(TAG, "%s failed to setup network, ret: %d", __func__, ret);
        goto err;
    }

    // ...

err:
    return ret;
}
```
其中，m5310a_net_attach进行模组初始化操作，在m5310a.c中定义。M5310A_SOCK_UNKNOWN为的sock type初始值，在m5310a.c中定义。


- **HAL_SAL_Deinit**
  - 释放相关系统资源；
  - 重置AT模组（可选）。
  - 以m5310a模组为例，下面是m5310a模组的HAL_SAL_Deinit接口的实现片段：

```c
static int HAL_SAL_Deinit(void)
{
    if (!inited) {
        return 0;
    }

    /* 释放锁、信号量等资源。 */
    if (aos_mutex_is_valid(&g_link_mutex)) {
        aos_mutex_free(&g_link_mutex);
    }

    // ...

    return 0;
}
```

- **HAL_SAL_Start**
  - 操作AT模组，建立Socket连接。对接时，如果有需要，注意维护文件描述符（FD）与模组连接号（link ID）或其他表示Socket连接的平台特征的映射关系。
  - 注意，该接口需要等待socket连接成功（或出现错误）时返回，一般是在发出连接AT命令后，需要使用信号量方式，等待连接成功的事件通知（如果有）。
  - 最少需要支持`TCP Client`功能，可选的功能支持包括`TCP Server`、`UDP单播`、`UDP多播`、`SSL Client`。
  - 以m5310a模组为例，下面是m5310a模组的HAL_SAL_Start接口的实现片段：

```c
static int HAL_SAL_Start(sal_conn_t *conn)
{
    // ...
 
    /* 上层socket fd和AT层fd的转换 */
    for (link_id = 0; link_id < LINK_ID_MAX; link_id++) {
        if (g_link[link_id].fd >= 0) {
            continue;
        } else {
            g_link[link_id].fd = conn->fd;
            break;
        }
    }

    // ...

    /* 根据接口参数，设置AT层对应的连接协议和类型，支持TCP Client和UDP Unicast */
    int protocol, port = conn->r_port;
    char *type, *ip = conn->addr, *p;
    switch (conn->type) {
        case TCP_CLIENT:
            protocol = 6;
            type = "STREAM";
            g_link[link_id].socktype = M5310A_SOCK_TCPCLIENT;
            break;
        case UDP_UNICAST:
            protocol = 17;
            type = "DGRAM";
            g_link[link_id].socktype = M5310A_SOCK_UDPCLIENT;
            break;
        case SSL_CLIENT:
        case TCP_SERVER:
        case UDP_BROADCAST:
            M5310A_LOGW(TAG, "Connection type %d not supported.", conn->type);
            goto err;
        default:
            M5310A_LOGE(TAG, "%s %d invalid c->type %d\r\n", __func__, __LINE__, conn->type);
            goto err;
    }

    /* 发送“AT+NSOCR”AT命令，创建socket */
    snprintf(cmd, sizeof(cmd) - 1, "%s=%s,%d", M5310A_AT_CMD_SOCKET_CREATE, type, protocol);
    M5310A_SEND_AT(cmd, rsp);
    if (strstr(rsp, M5310A_AT_CMD_SUCCESS_RSP) == NULL) {
        M5310A_LOGE(TAG, "%s %d failed rsp %s\r\n", __func__, __LINE__, rsp);
        goto err;
    }

    /* 获取和存储AT层fd信息。 */
    p = rsp;
    while (*p > '9' || *p < '0') p++;
    g_link[link_id].at_fd = *p - '0';

    /* 对于UDP类型连接，无需connect，从这里返回。 */
    if (protocol == 17) goto success;

    /*
     * Get socket connect lock so as to avoid 2 or more
     * connect AT cmds entangled with each other.
     *
     * @note Be careful of deadlock.
     */
    if (aos_mutex_lock(&g_connect_mutex, AOS_WAIT_FOREVER) != 0) {
        M5310A_LOGE(TAG, "Failed to lock mutex (%s).", __func__);
        goto err;
    }

    /* 对于TCP类型连接，执行“AT+NSOCO”AT命令，进行AT层connect操作。 */
    memset(cmd, 0, sizeof(cmd));
    memset(rsp, 0, sizeof(rsp));
    snprintf(cmd, sizeof(cmd) - 1, "%s=%d,%s,%d",
             M5310A_AT_CMD_SOCKET_CONNECT,
             g_link[link_id].at_fd, ip, port);
    M5310A_SEND_AT(cmd, rsp);
    if (strstr(rsp, M5310A_AT_CMD_SUCCESS_RSP) == NULL) {
        M5310A_LOGE(TAG, "%s %d failed rsp %s\r\n", __func__, __LINE__, rsp);
        goto err;
    }

    /* 等待连接成功（或超时）后才能返回。 */
    if (aos_sem_is_valid(&g_sem_connect)) {
        if (aos_sem_wait(&g_sem_connect, SEM_WAIT_DURATION_MS) != 0) {
            M5310A_LOGE(TAG, "Failed to wait sem %s %d", __func__, __LINE__);
            aos_mutex_unlock(&g_connect_mutex);
            goto err;
        }
    }

    aos_mutex_unlock(&g_connect_mutex);

    M5310A_LOGD(TAG, "%s sem_wait succeed.", __func__);

success:
    aos_mutex_unlock(&g_link_mutex);
    return 0;

err:
    g_link[link_id].fd = -1;
    g_link[link_id].at_fd = -1;
    g_link[link_id].socktype = M5310A_SOCK_UNKNOWN;
    aos_mutex_unlock(&g_link_mutex);
    return -1;
}
```

- **HAL_SAL_Close**
  - 操作AT模组，关闭Socket连接服务，释放link ID等资源（如果有需要）。该接口需要接收到socket关闭成功的通知（如果有）后才能返回。
  - 以m5310a模组为例，下面是m5310a模组的HAL_SAL_Close接口的实现片段：

```c
static int HAL_SAL_Close(int fd, int32_t remote_port)
{
    // ...
    
    /* AT层fd和上层fd转换 */
    link_id = fd2linkid(fd);
    if (link_id >= 0) {
        /* 执行“AT+NSOCL”AT命令关闭连接。 */
        at_fd = g_link[link_id].at_fd;
        snprintf(cmd, sizeof(cmd) - 1, "%s=%u",
                 M5310A_AT_CMD_SOCKET_CLOSE, at_fd);
        M5310A_SEND_AT(cmd, rsp);
        if (strstr(rsp, M5310A_AT_CMD_SUCCESS_RSP) == NULL) {
            M5310A_LOGE(TAG, "%s %d failed rsp %s\r\n", __func__, __LINE__, rsp);
            return -1;
        }
        
        // ...

        /* 重置g_link状态信息。 */
        g_link[link_id].fd = -1;
        g_link[link_id].fd = -1;
        g_link[link_id].socktype = M5310A_SOCK_UNKNOWN;

        // ...
    }

    return link_id < 0 ? -1 : 0;
}
```

- **HAL_SAL_DomainToIp**
  - DNS解析服务，获取域名相关的服务器IP地址（返回类似`10.102.70.17`这种IP地址字符串）。如果DNS结果嵌在reply_success_postfix之前返回，则驱动中需要解析命令回复字段解析到DNS结果；如果DNS结果以事件方式通知，则需要以事件方式处理和解析。
  - 以m5310a模组为例，下面是m5310a模组的HAL_SAL_DomainToIp接口的实现片段：

```c
static int HAL_SAL_DomainToIp(char *domain, char ip[16])
{
    // ...

    /* 执行“AT+CMDNS”AT命令 */
    snprintf(cmd, sizeof(cmd) - 1, "%s=\"%s\"",
             M5310A_AT_CMD_DNS, domain);
    M5310A_SEND_AT(cmd, rsp);
    if (strstr(rsp, M5310A_AT_CMD_SUCCESS_RSP) == NULL) {
        M5310A_LOGE(TAG, "%s %d failed rsp %s\r\n", __func__, __LINE__, rsp);
        goto err;
    }

    /* 等待DNS结果（或超时）后再返回。 */
    if (aos_sem_wait(&g_sem_dns, SEM_WAIT_DURATION_MS) != 0) {
        M5310A_LOGE(TAG, "%s sem_wait failed", __func__);
        goto err;
    }

    // ...
}
```

- **HAL_SAL_Send**
  - 发送Socket数据。对接时，注意转换文件描述符FD到对应的Socket连接号，并通过相应的AT命令进行数据发送操作。注意，有些模组以hex格式发送数据，而有些模组以hexstring格式发送，注意转换。
  - 注意：需要保存HAL_SAL_Start接口传入的参数信息，用以判断是否UDP协议及远端IP、端口的值，不能使用本接口中的remote_ip等参数。
  - 以m5310a模组为例，下面是m5310a模组的HAL_SAL_Send接口的实现片段：

```c
static int HAL_SAL_Send(int fd,
                 uint8_t *data,
                 uint32_t len,
                 char remote_ip[16],
                 int32_t remote_port,
                 int32_t timeout)
{
    // ...

    /* 上层fd映射诚link_id和AT fd。 */
    link_id = fd2linkid(fd);
    atfd = g_link[link_id].at_fd;

    /* 根据g_link中保存的信息判断是TCP还是UDP发送。 */
    if (g_link[link_id].socktype == M5310A_SOCK_UDPCLIENT) { /* UDP send */
        /* 使用“AT+NSOST”AT命令进行UDP数据发送 */
        snprintf(cmd, cmdlen - 1, "%s=%d,%s,%d,%d,", M5310A_AT_CMD_UDP_SEND,
                 atfd, g_link[link_id].remote_ip, g_link[link_id].remote_port, len);
        hex2hexstr(cmd + strlen(cmd), data, len);
        /* This is a trick, anyway it works. */
        snprintf(cmd + strlen(cmd), 1, "%s", "\n");
        M5310A_SEND_AT_NO_REPLY(cmd, false);
    } else if (g_link[link_id].socktype == M5310A_SOCK_TCPCLIENT) { /* TCP send */
        /* 使用“AT+NSOSD”AT命令进行UTCP数据发送 */
        snprintf(cmd, cmdlen - 1, "%s=%d,%d,", M5310A_AT_CMD_TCP_SEND, atfd, len);
        hex2hexstr(cmd + strlen(cmd), data, len);
        M5310A_SEND_AT(cmd, rsp);
        // ...
    } else {
        /* TODO */
    }

    // ...
}
```

- **HAL_SAL_RegisterNetconnDataInputCb**
  - 注册数据接收回调函数。该回调函数需要在AT层接收到socket数据时被调用。具体调用方式请参考示例驱动代码。
  - 以m5310a模组为例，下面是m5310a模组的HAL_SAL_RegisterNetconnDataInputCb接口的实现片段：

```c
/* 此处注册回调 */
static int HAL_SAL_RegisterNetconnDataInputCb(netconn_data_input_cb_t cb)
{
    if (cb) {
        /* 通过全局变量g_netconn_data_input_cb保存回调函数指针。 */
        g_netconn_data_input_cb = cb;
    }

    return 0;
}

/* 在接收处理函数中，调用回调上报数据 */
static void m5310a_socket_data2_event_handler(void *arg, char *buf, int buflen)
{
    // ...
    
    /* We use static mm skt_recv_buffer here to avoid frequent mm alloc */
    while (left > 0) {
        // ...

        /* 调用回调函数进行数据通知 */
        if (g_netconn_data_input_cb) {
            /* TODO get recv data src ip and port*/
            if (g_netconn_data_input_cb(fd, skt_recv_buffer, toread, NULL, 0)) {
                M5310A_LOGE(TAG, " %s socket %d get data len %d fail to post"
                            " to sal, drop it\n", __func__, fd, toread);
            }
        }

        // ...
    }
}
```
在进行新模组驱动HAL对接时，可以参考AliOS Things源码中提供的示例模组驱动代码，结合上述介绍，完成驱动HAL接口的实现和对接。

## 4.4 atparser使用指南
在对接HAL的过程中，对AT模组的收发操作，建议借助atparser模块进行。atparser模块是AliOS Things中为方便AT命令的收发处理，抽象的一套AT处理机制。atparser提供一个统一的线程进行串口AT数据的接收，对于AT数据的发送，atparser提供了发送接口，用户可以异步进行发送，由atparser的接收线程负责接收并将AT回复反馈给对应发送者。同时atparser提供了一套事件处理机制。

借助atparser，驱动对接用户不需要太关注AT通道的底层收发操作，而专注在自身模组的AT命令处理上。

如果驱动的实现依赖atparser组件，请修改模组驱动的Makefile文件，添加atparser的组件依赖。m5310a完整的makefile示例如下：

```makefile
NAME := device_sal_m5310a

$(NAME)_MBINS_TYPE := kernel
$(NAME)_VERSION := 1.0.0
$(NAME)_SUMMARY := sal hal implmentation for m5310a
GLOBAL_DEFINES += DEV_SAL_M5310A

$(NAME)_COMPONENTS += atparser # 添加对atparser组件的依赖
$(NAME)_SOURCES += m5310a.c

GLOBAL_INCLUDES += ./ # 添加头文件暴露，使外部可以引用atcmd_config_module.h头文件
```

以下是atparser模块的用户接口和相关定义介绍（请参考`include/network/nal/atparser/atparser.h`文件）。用户可以参考其他示例驱动对atparser接口的使用，结合本文的介绍，使用atparser进行驱动实现。

- at_dev_type_t

AT设备类型定义，目前仅支持UART。如有需要，用户可以添加其他类型设备（如SPI）。

- at_reply_config_t

AT命令回复的特征配置，请使用`atcmd_config_module.h`文件中提供的定义进行填充。包括：reply_prefix，回复字串前缀（如有，一般为换行和回车“\r\n”）；reply_success_postfix，成功时的回复后缀（一般为“OK\r\n”）；reply_fail_postfix，失败时的回复后缀（一般为“ERROR\r\n”）。

- at_config_t

AT相关的配置，包括串口号、设备类型、设备配置参数（如串口波特率、校验位等）、AT命令回复特征、发送分隔符、AT命令发送超时、发送配置、接收线程配置等。

```c
typedef struct {
    uint8_t            port;                 /* dev port. Compulsory */
    at_dev_type_t      type;                 /* dev type. Compulsory */
    void               *dev_cfg;             /* dev config. Compulsory. For UART, see uart_config_t in hal/uart.h */
    at_reply_config_t  reply_cfg;            /* AT receive prefix and postfix. Compulsory. */
    char               *send_delimiter;      /* AT sending delimiter between command and data. Optional, "\r" */
    uint32_t           timeout_ms;           /* AT send or receive timeout in millisecond. Optional, 1000 ms by defaut */
    uint8_t            send_wait_prompt;     /* whether AT send waits prompt before sending data. Optional, 0 by default */
    uint32_t           prompt_timeout_ms;    /* AT send wait prompt timeout in millisecond. Optional, 200 ms by default */
    uint8_t            send_data_no_wait;    /* whether AT send waits response after sending data. Optional, 0 by default */
    int                recv_task_priority;   /* AT recv task priority. Optional, 32 by default*/
    int                recv_task_stacksize;  /* AT recv task stacksize. Optional, 1K by default */
} at_config_t;
```

- at_recv_cb

**接口描述**：AT事件回调处理函数原型定义。


**参数含义**：arg - 注册回调时传入的arg参数；buf - 注册回调时传入的recvbuf参数，用于存储有prefix和postfix特征匹配的结果；buflen - buf中接收到的内容的实际长度。


**返回值**：空。

- at_init

**接口描述**：初始化atparser核心功能，在HAL对接中add_dev接口开始处调用。


**参数含义**：无。


**返回值**：0：成功，-1：失败。

- at_deinit

**接口描述**：销毁atparser模块（注销AT设备等操作），驱动中一般不调用。


**参数含义**：arg - 注册回调时传入的arg参数；buf - 注册回调时传入的recvbuf参数，用于存储有prefix和postfix特征匹配的结果；buflen - buf中接收到的内容的实际长度。


**返回值**：0：成功，-1：失败。

- at_add_dev

**接口描述**：添加AT设备，在HAL对接中add_dev接口中调用。


**参数含义**：config - 设备相关的配置。


**返回值**：大于等于0：设备句柄，-1：添加失败。

- at_delete_dev

**接口描述**：删除AT设备。一般驱动不用调用。


**参数含义**：AT设备的句柄。


**返回值**：0：成功，-1：失败。

- at_send_wait_reply

**接口描述**：发送AT命令及数据，并等待回复。该函数阻塞，直到接收到该AT命令的回复，或者发送等待超时（默认是5s，用户可配置）。


**参数含义**：fd - AT设备句柄；cmd - 需要发送的AT命令字符串；cmdlen - AT命令长度；delimiter - 发送AT命令与数据时二者之间的分隔符（如果不适用，请填充为NULL）；data - 需要发送的数据内容；datalen - 数据长度；replybuf - 回复结果接收缓存；bufsize - 接收缓存的长度；atcmdconfig - AT命令回复的特征配置。


**返回值**：0：成功，-1：失败。


**注意**：（1）命令的回复中有reply_success_postfix（或reply_fail_postfix）特征，才能调用此函数进行发送；（2）发送AT命令后，reply_success_postfix（或reply_fail_postfix）及之前的串口数据会被认定为AT命令的回复。如果你的模组在reply_success_postfix（或reply_fail_postfix）还有内容回复，一般建议当作AT事件处理（在Init中注册相应的回调函数进行处理）。

- at_send_no_reply

**接口描述**：发送AT命令及数据，无需等待回复。该函数非阻塞，发送到设备缓冲区即返回。


**参数含义**：fd - AT设备句柄；data - 需要发送的命令或数据；datalen - 命令或数据长度；delimiter - 是否发送分隔符（`at_config_t`结构体中定义的`send_delimiter`）。


**返回值**：0：成功，-1：失败。


注意：该函数一般在命令或数据发送后没有回复的情况下调用。对于有回复、但是回复中不带reply_success_postfix（或reply_fail_postfix）特征的命令和数据发送，一般也需要调用此函数进行发送，并注册相应的事件回调函数对回复进行处理。

- at_read

**接口描述**：读取AT设备。


**参数含义**：fd - AT设备句柄；outbuf - 结果缓存；readsize - 缓存大小。


**返回值**：-1：读取失败，>0：读取到的字节长度。

- at_register_callback

**接口描述**：注册AT事件回调处理函数。一般不在AT命令回复特征段内的数据，都需要按照事件进行处理，如网络连接成功通知、Socket数据到达通知等事件。


**参数含义**：fd - AT设备句柄；prefix - 事件前缀标记，必选项，如m5310a模组的Socket数据通知事件“+NSONMI”、mk3060模组的WiFi事件通知“+WEVENT”等；postfix - 事件后缀标记，可选项，如果不使用，请填充为NULL；recvbuf - 匹配同时有prefix和postfix特征的回复时的结果缓存区，可选项，如果不适用请填充为NULL；bufsize - recvbuf缓存区大小，如果recvbuf为NULL时该参数填充为0；cb - 事件回调处理函数，必选项；arg - 提供给事件回调处理函数的参数，可选项，如果没有请填充为NULL。


**返回值**：0：成功，-1：失败。

## 4.5 HAL对接特别提醒
以下是在进行HAL对接的过程中，需要特别注意的一些事项：

- AT事件的认定和处理

一般位于命令发送和reply_success_postfix（一般是“OK\r\n”）或reply_fail_postfix（一般是“ERROR\r\n”）字段之间的内容，认定为命令的回复；而位于reply_success_postfix（或reply_fail_postfix）之后的内容，需要以事件方式进行处理。对于某些模组，事件可能不是“+XXX“的形式出现，这种情况很容易被误解为不是事件，如m5310a模组的网络连接事件“CONNECT OK”，建议当作事件进行处理。

- 在事件处理中不能调用at_send_wait_reply接口

因为事件处理函数是在atparser接收线程上下文中，这时调用at_send_wait_reply需要等待atparser接收线程处理命令的返回，会造成atparser接收线程线程卡死。

- UART缓存大小的问题说明

在使用串口设备外接AT模组时，需要注意设置合适的串口接收缓存（如ringbuf）的大小。如果缓存太小，在接受较大的串口数据时容易溢出/新覆盖旧数据的问题，影响AT结果的解析。

- AT命令发送超时等待设置

默认AT命令发出后，回复（如果有）应该在5秒内返回，一般命令的回复都可以在5秒内返回。如果该模组有某些AT命令需要等到较长时间，则需要注意加大AT等待命令回复的timeout时间（文件network/nal/atparser/atparser_opts.h中的AT_TASK_DEFAULT_WAIT_TIME_MS宏配置），或者避免使用at_send_wait_reply进行这种命令的回复等待。例如，m5310a模组的 “AT+NRB”。

- HAL_SAL_Send接口实现时UDP类型的判定

不能依靠remote_port/remote_ip参数是否有效来来判断是TCP还是UDP，只能依赖fd。可以在socket建立时，驱动可以记录该连接的类型（TCP还是UDP），然后在socket发送/接收时使用该记录进行类型判断。

- 调用sal_ad_dev添加串口设备的问题

参数中的name需与驱动名（一般也是模组名）相匹配，config除port和timeout外，其他字段的值需要使用hal/uart.h中的宏定义，需添加atcmd_config_module.h头文件引用。

- HAL_SAL_Send接口中UDP远端IP和端口的问题

remote_ip、remote_port参数针对UDP client无效，对接时需注意，远端ip/port参数需要从HAL_SAL_Start接口中自己保存，并在发送时查询使用。

- AT命令结束符的差异处理

例如，大部分AT命令以"\r\n"结尾，但可能存在少数以"\n"结尾（如m5310a的AT+NSOST）的命令。如果选择了以“\r\n”为通用的命令结尾符，则需要对以“\n”结束的AT命令进行特别操作。

# 5 驱动集成
## 5.1 menuconfig集成。

- 修改driver/sal目录下该模组类型目录下的Config.in文件（以m5310a为例，应修改drivers/sal/nbiot/Config.in文件）。根据已有示例添加该模组相关的行（即以下示例中m5310a所在的行）。

```shell
config AOS_SAL_NBIOT
    bool "NBIOT"

if AOS_SAL_NBIOT

choice
    prompt "Device"
    default AOS_COMP_DEVICE_SAL_M5310A if BSP_EXTERNAL_NBIOT_MODULE = "nbiot.m5310a"

config AOS_NBIOT_NULL
    bool "Null"

source "drivers/sal/nbiot/m5310a/Config.in"

endchoice

endif
```

- 修改driver/Config.in文件，添加模组类型的支持（参考下列diff文件的修改，如果是在已有类型上添加模组，则该步骤可以省略）。

```diff
diff --git a/drivers/Config.in b/drivers/Config.in
index 0f6c3a3..a21a7c4 100644
--- a/drivers/Config.in
+++ b/drivers/Config.in
@@ -7,10 +7,12 @@ choice
     prompt "SAL Drivers Configuration"
     default AOS_SAL_WIFI if BSP_EXTERNAL_WIFI_MODULE != ""
     default AOS_SAL_LTE if BSP_EXTERNAL_LTE_MODULE != ""
+    default AOS_SAL_NBIOT if BSP_EXTERNAL_NBIOT_MODULE != ""

     source "drivers/sal/wifi/Config.in"
     source "drivers/sal/gprs/Config.in"
     source "drivers/sal/lte/Config.in"
+    source "drivers/sal/nbiot/Config.in"
 endchoice
 endif

```

## 5.2 C源码集成

- 在SAL Core中增加模组的设备初始化函数调用。参考以下修改：

```diff
diff --git a/network/nal/sal/src/sal_sockets.c b/network/nal/sal/src/sal_sockets.c
index 4c0f220..5f94213 100644
--- a/network/nal/sal/src/sal_sockets.c
+++ b/network/nal/sal/src/sal_sockets.c
@@ -1621,6 +1621,13 @@ int sal_module_register(sal_op_t *module)
 int sal_device_init()
 {
    int ret = 0;
+#ifdef DEV_SAL_M5310A
+   if(m5310a_sal_device_init() != 0)
+   {
+       SAL_ERROR("m5310a sal init failed\n");
+       return -1;
+   }
+#endif
 #ifdef DEV_SAL_BK7231
    if(bk7231_sal_device_init() != 0)
    {
```

**备注**：由于m5310a模组支持的OOB事件超过我们3.0版本AT目前支持的个数（默认是5），因此跑m5310a模组时还要修改AT OOB的个数限制（后续版本会同步更新，不用手动修改）。参考下列修改：

```diff
diff --git a/network/nal/atparser/atparser_internal.h b/network/nal/atparser/atparser_internal.h
index 31dc215..df100cd 100644
--- a/network/nal/atparser/atparser_internal.h
+++ b/network/nal/atparser/atparser_internal.h
@@ -9,7 +9,7 @@

 #include "atparser_opts.h"

-#define OOB_MAX 5
+#define OOB_MAX 10

 typedef enum {
     AT_RSP_WAITPROMPT = 0,
```

## 5.3 Makefile集成

- 将新对接模组加入SAL设备组件编译列表，修改`network/nal/sal/aos.mk`文件，m5310a的示例如下：

```diff
diff --git a/network/nal/sal/aos.mk b/network/nal/sal/aos.mk
index 1f5ce93..fa44fbf 100644
--- a/network/nal/sal/aos.mk
+++ b/network/nal/sal/aos.mk
@@ -27,4 +27,6 @@ else ifeq (wifi.esp8266,$(module))
 $(NAME)_COMPONENTS += device_sal_esp8266
 else ifeq (wifi.athost,$(module))
 $(NAME)_COMPONENTS += device_sal_athost
+else ifeq (nbiot.m5310a,$(module))
+$(NAME)_COMPONENTS += device_sal_m5310a
 endif
```

## 5.4 板级集成

- 修改MCU侧board的makefile文件，增加新对接模组的支持。以下以stm32f103rb-nucleo为例：

```diff
diff --git a/board/stm32f103rb-nucleo/aos.mk b/board/stm32f103rb-nucleo/aos.mk
index 2858ece..e8e8d5e 100644
--- a/board/stm32f103rb-nucleo/aos.mk
+++ b/board/stm32f103rb-nucleo/aos.mk
@@ -23,7 +23,12 @@ ywss_support ?= 0
 GLOBAL_DEFINES += CONFIG_NO_TCPIP

 #depends on sal module if select sal function via build option "AOS_NETWORK_SAL=y"
-AOS_NETWORK_SAL        ?= n
+AOS_NETWORK_SAL        ?= y
+ifeq (y,$(AOS_NETWORK_SAL))
+$(NAME)_COMPONENTS += sal netmgr
+module             ?= nbiot.m5310a
+else
+GLOBAL_DEFINES += CONFIG_NO_TCPIP
+endif
+

 ifeq ($(COMPILER), armcc)
 $(NAME)_SOURCES += startup/startup_stm32f103xb_keil.s
```

**备注**：对于已对接过SAL模组的board（例如developerkit），其makefile中已有SAL相关的支持，只需求修改module那一行即可。

- 修改MCU侧board的menuconfig配置文件，增加对新对接模组的支持。以下以stm32f103rb-nucleo为例：

```diff
diff --git a/board/stm32f103rb-nucleo/Config.in b/board/stm32f103rb-nucleo/Config.in
index 2ff9918..ff8f919 100644
--- a/board/stm32f103rb-nucleo/Config.in
+++ b/board/stm32f103rb-nucleo/Config.in
@@ -3,10 +3,20 @@ config AOS_BOARD_STM32F103RB_NUCLEO
     select AOS_MCU_STM32F1XX
     select AOS_COMP_KERNEL_INIT
     select AOS_COMP_OSAL_AOS
+    select AOS_COMP_SAL if AOS_NETWORK_SAL
     help
         **The Stm32f103rb-nucleo** performance line microcontroller incorporates the high-performance ARM® Cortex™-M3 32-bit RISC core operating at a 72 MHz frequency, high-speed embedded memories, and an extensive range of enhanced I/Os and peripherals connected to two APB buses. The STM32F103RC offers three 12-bit ADCs, four general-purpose 16- bit timers plus two PWM timers, as well as standard and advanced communication interfaces: up to two I2Cs, three SPIs, two I2Ss, one SDIO, five USARTs, an USB and a CAN.

 if AOS_BOARD_STM32F103RB_NUCLEO
 # Configurations for board stm32f103rb-nucleo

+config BSP_SUPPORT_EXTERNAL_MODULE
+    bool
+    default y
+
+config BSP_EXTERNAL_NBIOT_MODULE
+    string
+    depends on BSP_SUPPORT_EXTERNAL_MODULE
+    default "nbiot.m5310a"
+
 endif
```

**备注**：对于已对接过SAL模组的board（例如developerkit），其menuconfig配置文件中已有SAL相关的支持，只需求修改模组类型和模组名信息相关的行即可，参考以下修改：

```diff
diff --git a/board/developerkit/Config.in b/board/developerkit/Config.in
index d38c1ad..9fd4689 100644
--- a/board/developerkit/Config.in
+++ b/board/developerkit/Config.in
@@ -62,9 +62,9 @@ config BSP_SUPPORT_EXTERNAL_MODULE
     bool
     default y

-config BSP_EXTERNAL_WIFI_MODULE
+config BSP_EXTERNAL_NBIOT_MODULE
     string
     depends on BSP_SUPPORT_EXTERNAL_MODULE
-    default "wifi.bk7231"
+    default "nbiot.m5310a"

 endif
```

# 6 驱动使用与验证
本章以在stm32f103rb-nucleo（MCU侧）使用m5310a外挂模组为例，展示如何使用和测试新对接模组的SAL驱动。其他MCU板子和模组的使用步骤类似。
## 6.1 硬件准备
本文的介绍基于stm32f103rb-nucleo（MCU） + m5310a（NBIoT模组），请参考下列步骤准备好硬件环境：

- 通过杜邦线给m5310a的供电（参考下图）。**备注**：可以通过MCU侧的5V和GND给m5310a供电，但是建议通过独立电源给m5310a供电，因为m5310a的启动和初始化需要较长时间。
- 通过杜邦线连接MCU和模组的AT串口收发信号线（RX/TX交叉连接，参考下图）。
- 通过USB线给MCU供电（参考下图）。

![接线示例](https://img.alicdn.com/tfs/TB1jvlJvqL7gK0jSZFBXXXZZpXa-4032-3024.jpg)


## 6.2 测试app开发/添加
用户可以使用本章最后附件中的测试用例（app/example/sal_simple_test），也可以自己开发一个使用外挂模组进行socket通信的测试用例。

请参考以下步骤，将测试用例添加到AliOS Things工程中。

- 集成测试用例。修改app/example/Config.in，增加测试用例的入口。

```diff
diff --git a/app/example/Config.in b/app/example/Config.in
index 7176dc5..0e5c96e 100644
--- a/app/example/Config.in
+++ b/app/example/Config.in
@@ -252,5 +252,11 @@ if AOS_APP_UAPP2
         default "uapp2"
 endif

+source "app/example/sal_simple_test/Config.in"
+if AOS_APP_SAL_SIMPLE_TEST
+    config AOS_BUILD_APP
+        default "sal_simple_test"
+endif
+
 endchoice
 endif
```

## 6.3 在app中完成模组设备的添加
如果使用附件中的测试用例，则该步骤可省略。

如果使用自己开发的用例，则需要在app中完成模组设备的添加操作。请参考以下代码对模组设备进行添加操作（其中串口号是指MCU board用于外挂模组的串口号，参考下一章节中board相关的配置）：

```c
    /**
     * MCU board‘s AT uart port number:
     *    - 1
     * @note subject to change according to your case.
     *
     * module uart config:
     *    - 9600
     *    - 8n1
     *    - no flow control
     *    - tx/rx mode
     * @note These configs are subject to change according to the module you use.
     */
    data.uart_dev.port = 1;
    data.uart_dev.config.baud_rate = 9600;
    data.uart_dev.config.data_width = DATA_WIDTH_8BIT;
    data.uart_dev.config.parity = NO_PARITY;
    data.uart_dev.config.stop_bits  = STOP_BITS_1;
    data.uart_dev.config.flow_control = FLOW_CONTROL_DISABLED;
    data.uart_dev.config.mode = MODE_TX_RX;

    /**
     * Register and Initialize SAL device.
     *
     * Attention:
     *     (1) If register SAL device using "sal_add_dev(NULL, NULL)",
     *         it means to use the device and related configuration
     *         prodived in board settings (e.g. module ?= wifi.bk7231
     *         in board/developer/aos.mk).
     *     (2) If want to register SAL device with specific parameters,
     *         Please use 'sal_add_dev("<device_name>", "<dev_para>")',
     *         in which the <device_name> is the module name of the AT device,
     *         and fill in the <dev_para> with UART config parameters.
     */
    if (sal_add_dev("m5310a", &data) != 0) {
        LOG("Failed to add SAL device!");
        return -1;
    }
```

## 6.4 使能board外挂模组的串口
MCU board为了支持外挂模组通信，需要指定串口设备号，并对串口进行相应的初始化。以下以stm32f103rb-nucleo为例进行说明和示例：

- 在board目录下添加AT串口设置文件atcmd_config_platform.h，在该文件中通过AT_UART_PORT宏指定AT串口设备号（这里使用串口号1，对应USART3）。

```c
/*
 * Copyright (C) 2015-2017 Alibaba Group Holding Limited
 */

#ifndef _ATCMD_CONFIG_PLATFORM_H_
#define _ATCMD_CONFIG_PLATFORM_H_

// AT uart
#define AT_UART_PORT 1

#endif
```

- 修改board相关源码，对串口1进行初始化设置。

```diff
diff --git a/board/stm32f103rb-nucleo/config/board.h b/board/stm32f103rb-nucleo/config/board.h
index 650d217..a524ea8 100644
--- a/board/stm32f103rb-nucleo/config/board.h
+++ b/board/stm32f103rb-nucleo/config/board.h
@@ -61,6 +61,7 @@ extern "C" {

 typedef enum{
     PORT_UART_STD,
+    PORT_UART_AT,
     PORT_UART_SIZE,
     PORT_UART_INVALID = 255,
 }PORT_UART_TYPE;
@@ -86,6 +87,10 @@ void Error_Handler(void);
 #define USART_TX_GPIO_Port GPIOA
 #define USART_RX_Pin GPIO_PIN_3
 #define USART_RX_GPIO_Port GPIOA
+#define USART3_TX_Pin GPIO_PIN_10
+#define USART3_TX_GPIO_Port GPIOC
+#define USART3_RX_Pin GPIO_PIN_11
+#define USART3_RX_GPIO_Port GPIOC
 #define LD2_Pin GPIO_PIN_5
 #define LD2_GPIO_Port GPIOA
 #define TMS_Pin GPIO_PIN_13
diff --git a/board/stm32f103rb-nucleo/startup/board.c b/board/stm32f103rb-nucleo/startup/board.c
index f942425..51b17c8 100644
--- a/board/stm32f103rb-nucleo/startup/board.c
+++ b/board/stm32f103rb-nucleo/startup/board.c
@@ -46,7 +46,8 @@ DMA_HandleTypeDef hdma_usart2_tx;
 DMA_HandleTypeDef hdma_usart2_rx;
 UART_MAPPING UART_MAPPING_TABLE[] =
 {
-  {PORT_UART_STD, USART2, {UART_OVERSAMPLING_16, 64}}
+  {PORT_UART_STD, USART2, {UART_OVERSAMPLING_16, 64}},
+  {PORT_UART_AT, USART3, {UART_OVERSAMPLING_16, 256}},
 };

 static void stduart_init(void)
 diff --git a/board/stm32f103rb-nucleo/drivers/stm32f1xx_hal_msp.c b/board/stm32f103rb-nucleo/drivers/stm32f1xx_hal_msp.c
index fb1a916..8e856be 100644
--- a/board/stm32f103rb-nucleo/drivers/stm32f1xx_hal_msp.c
+++ b/board/stm32f103rb-nucleo/drivers/stm32f1xx_hal_msp.c
@@ -171,6 +171,77 @@ void HAL_UART_MspInit(UART_HandleTypeDef* huart)
   /* USER CODE BEGIN USART2_MspInit 1 */

   /* USER CODE END USART2_MspInit 1 */
+  }  else if (huart->Instance == USART3) {
+  /* USER CODE BEGIN USART3_MspInit 0 */
+
+  /* USER CODE END USART3_MspInit 0 */
+    /* Peripheral clock enable */
+    __HAL_RCC_USART3_CLK_ENABLE();
+
+    __HAL_RCC_GPIOC_CLK_ENABLE();
+    /**USART3 GPIO Configuration
+    PC10     ------> USART3_TX
+    PC11     ------> USART3_RX
+    */
+
+#ifdef STM32F103_ENABLE_UART_DMA
+    GPIO_InitStruct.Pin = USART3_TX_Pin|USART3_RX_Pin;
+    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
+    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_LOW;
+    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
+
+    /* USART3 DMA Init */
+    /* USART3_TX Init */
+    hdma_usart3_tx.Instance = DMA1_Channel5;
+    hdma_usart3_tx.Init.Direction = DMA_MEMORY_TO_PERIPH;
+    hdma_usart3_tx.Init.PeriphInc = DMA_PINC_DISABLE;
+    hdma_usart3_tx.Init.MemInc = DMA_MINC_ENABLE;
+    hdma_usart3_tx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
+    hdma_usart3_tx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
+    hdma_usart3_tx.Init.Mode = DMA_NORMAL;
+    hdma_usart3_tx.Init.Priority = DMA_PRIORITY_LOW;
+    if (HAL_DMA_Init(&hdma_usart3_tx) != HAL_OK)
+    {
+      Error_Handler();
+    }
+
+    __HAL_LINKDMA(huart,hdmatx,hdma_usart2_tx);
+
+    /* USART3_RX Init */
+    hdma_usart3_rx.Instance = DMA1_Channel4;
+    hdma_usart3_rx.Init.Direction = DMA_PERIPH_TO_MEMORY;
+    hdma_usart3_rx.Init.PeriphInc = DMA_PINC_DISABLE;
+    hdma_usart3_rx.Init.MemInc = DMA_MINC_ENABLE;
+    hdma_usart3_rx.Init.PeriphDataAlignment = DMA_PDATAALIGN_BYTE;
+    hdma_usart3_rx.Init.MemDataAlignment = DMA_MDATAALIGN_BYTE;
+    hdma_usart3_rx.Init.Mode = DMA_NORMAL;
+    hdma_usart3_rx.Init.Priority = DMA_PRIORITY_LOW;
+    if (HAL_DMA_Init(&hdma_usart3_rx) != HAL_OK)
+    {
+      Error_Handler();
+    }
+
+    __HAL_LINKDMA(huart,hdmarx,hdma_usart3_rx);
+#else /* STM32F103_ENABLE_UART_DMA */
+    GPIO_InitStruct.Pin = GPIO_PIN_10;
+    GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
+    GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_HIGH;
+    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
+
+    GPIO_InitStruct.Pin = GPIO_PIN_11;
+    GPIO_InitStruct.Mode = GPIO_MODE_INPUT;
+    GPIO_InitStruct.Pull = GPIO_NOPULL;
+    HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);
+
+    __HAL_AFIO_REMAP_USART3_PARTIAL();
+#endif /* STM32F103_ENABLE_UART_DMA */
+
+    /* USART3 interrupt Init */
+    HAL_NVIC_SetPriority(USART3_IRQn, 0, 0);
+    HAL_NVIC_EnableIRQ(USART3_IRQn);
+  /* USER CODE BEGIN USART3_MspInit 1 */
+
+  /* USER CODE END USART3_MspInit 1 */
   }

 }
```

## 6.5 配置、编译和运行

- menuconfig配置

```shell
$ aos make distclean
$ aos make sal_simple_test@stm32f103rb-nucleo -c config
$ aos make menuconfig # 可选
```

- 编译固件

```shell
$ aos make clean; aos make
```

编译好的固件位于`out/sal_simple_test@stm32f103rb-nucleo/binary/`目录下。

- 烧录并运行固件

sal_simple_test测试用例将测试TCP和UDP功能，请注意测试日志输出。

# 7 Keil和IAR工程实践
用户可以通过以下命令生成Keil或IAR工程文件，然后就可以通过Keil或IAR工具进行工程的编译和调试了。

- 导出Keil工程

可以通过以下命令生成Keil工程，工程文件位于`projects/Keil/sal_simple_test@stm32f103rb-nucleo/keil_project/`目录下。

```shell
$ aos make export-keil # 生成Keil工程文件
```

- 导出IAR工程

可以通过以下命令生成IAR工程，工程文件位于`projects/IAR/sal_simple_test@stm32f103rb-nucleo/iar_project/`目录下。

```shell
$ aos make export-iar # 生成IAR工程文件
```

# 8 参考文档

# 9 总结
本文是AliOS Things 3.0 AT模组驱动一站式开发文档，涵盖了代码获取、驱动对接、驱动集成和验证等环节的详细步骤和示例。通过本文档，用户可以实现从零开始实现AT模组的对接和使用。

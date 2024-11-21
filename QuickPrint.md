**LeliCa通信协议文档（快捷打印版）**

[TOC]

# 修订记录

| 版本   | 时间       | 修订者 | 备注           |
| ------ | ---------- | ------ | -------------- |
| V1.0.0 | 2024-11-21 | YoChan | 协议初始化版本 |



# 概要

本通信协议由乐立卡为方便接入迷你打印机所指定。



# 一、快捷打印机识别

## 1、打印机蓝牙名称要求

迷你便携式打印机，一般使用蓝牙方式接入我们主端设备，所以主端识别靠识别蓝牙名称来进行区分相关机芯配置，如下表：

| 蓝牙名称 | 打印分辨率 | 打印点数 |
| :------: | :--------: | :------: |
|   P100   |   200DPI   | 384dots  |
|  P100S   |   300DPI   | 576dots  |



## 2、打印机需具备BLE服务

| BLE广播数据的厂商自定义数据字段（flag=0xFF） | 说明                         |
| -------------------------------------------- | ---------------------------- |
| 设备蓝牙地址                                 | 设备蓝牙地址方便主端进行连接 |


| Service                              | 说明                 |
| ------------------------------------ | -------------------- |
| 0000FF00-0000-1000-8000-00805F9B34FB | 用于设备通信的主服务 |

| Characteristic                       | Property                | 说明                                 |
| :----------------------------------- | :---------------------- | ------------------------------------ |
| 0000FF02-0000-1000-8000-00805F9B34FB | write(without response) | 基础数据下发                         |
| 0000FF03-0000-1000-8000-00805F9B34FB | notify                  | 蓝牙连接成功后，上报一次 Credit和MTU |



# 二、快捷打印模块

## 1、打印开始(LLC_PRN_START)

| 功能组 | 功能 |
| :----: | :--: |
|  0x1D  | 0x42 |

| 功能组 | 功能 | 数据格式 |  长度  | 打印速度 |
| :----: | :--: | :------: | :----: | :------: |
|  0x1F  | 0x28 |   0x70   | 2Bytes | 打印速度 |

| 功能组 | 功能 | 数据格式 |  长度  | 打印浓度 |
| :----: | :--: | :------: | :----: | :------: |
|  0x1F  | 0x28 |   0x73   | 2Bytes | 打印浓度 |

```
int q0_ptl_print_start(void)
{
    uint8_t printbuff[32] = {0};
    uint16_t printbuff_count = 0;

    /* 速度 */
    printbuff[printbuff_count++] = 0x1b;
    printbuff[printbuff_count++] = 0x42;
    printbuff[printbuff_count++] = 0x1F;
    printbuff[printbuff_count++] = 0x28;
    printbuff[printbuff_count++] = 0x73;
    printbuff[printbuff_count++] = 0x02;
    printbuff[printbuff_count++] = 0x00;
    printbuff[printbuff_count++] = 3;  /* 3mm/s */
    printbuff[printbuff_count++] = 0x00;
    /* 浓度 */
    printbuff[printbuff_count++] = 0x1F;
    printbuff[printbuff_count++] = 0x28;
    printbuff[printbuff_count++] = 0x73;
    printbuff[printbuff_count++] = 0x01;
    printbuff[printbuff_count++] = 0x00;
    printbuff[printbuff_count++] = 2;  /* dark */
    if (bsp_bt_send_data(printbuff, printbuff_count) != printbuff_count) {
        return -1;
    }
    return 0;
}
```





## 2、打印停止(LLC_PRN_STOP)

| 功能组 | 功能 | 空白走纸点行数 |
| :----: | :--: | :------------: |
|  0x1B  | 0x64 |      0x30      |

| 功能组 | 功能 | 打印结束 | 打印结束 |
| :----: | :--: | :------: | :------: |
|  0x1B  | 0x42 |   0x01   |   0x03   |

```
HOST发送实例

int q0_ptl_print_stop(uint32_t feedlines)
{
    uint8_t printbuff[7] = {0};

    printbuff[0] = 0x1b;
    printbuff[1] = 0x64;
    printbuff[2] = feedlines & 0xff;  /* 24dot/lines */
    printbuff[3] = 0x1b;
    printbuff[4] = 0x42;
    printbuff[5] = 0x01;
    printbuff[6] = 0x03;
    if (bsp_bt_send_data(printbuff, 7) != 7) {
        return -1;
    }
    return 0;
}
```



## 3、打印数据(LLC_PRN_DATA)

| 功能组 | 功能 | 数据格式 | 位图宽度 | 位图高度 |       打印数据       |
| :----: | :--: | :------: | :------: | :------: | :------------------: |
|  0x1D  | 0x76 |   0x30   |  2Bytes  |  2Bytes  | 打印位图点行数据1bit |

```
HOST发送实例

int q0_ptl_print_data(uint16_t linebytes, uint16_t offset, uint8_t *prn_data, uint16_t prn_datalen)
{
    uint8_t *printbuff = platform_mmu_alloc(prn_datalen + 25, sizeof(uint8_t));
    uint16_t printbuff_count = 0;
    platform_endian pkg_data_endian;

    if (printbuff == NULL) {
        return -1;
    }
    printbuff[printbuff_count++] = 0x1d;
    printbuff[printbuff_count++] = 0x76;
    printbuff[printbuff_count++] = 0x30;
    printbuff[printbuff_count++] = 0x00;
    pkg_data_endian.u32word = linebytes;               // 图像宽度
    printbuff[printbuff_count++] = pkg_data_endian.u32word_byte.lbyte_l;
    printbuff[printbuff_count++] = pkg_data_endian.u32word_byte.lbyte_h;
    pkg_data_endian.u32word = prn_datalen / linebytes; // 图像高度
    printbuff[printbuff_count++] = pkg_data_endian.u32word_byte.lbyte_l;
    printbuff[printbuff_count++] = pkg_data_endian.u32word_byte.lbyte_h;
    memcpy(&printbuff[printbuff_count], prn_data, prn_datalen);
    printbuff_count += prn_datalen;
    if (printbuff_count != bsp_bt_send_data(printbuff, printbuff_count)) {
        platform_mmu_free(printbuff);
        return -1;
    }
    platform_mmu_free(printbuff);
    return 0;
}
```






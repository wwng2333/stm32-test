# STM32G030 EEPROM 单字节读写实验任务书

**任务名称：** 基于 STM32G030 的 I²C EEPROM 单字节读写实验

**实验平台：** STM32G030F6P6

**使用外设：** UART、I²C1、EEPROM

**开发工具：** STM32CubeMX + Keil MDK（或现有 IDE）

**工程名称：** `G030_I2C_EEPROM_Test`

**任务要求：** 独立创建工程，分阶段调试，不直接复制旧工程代码。

## 一、实验目标

本次实验需要完成以下功能：

1. 使用 STM32CubeMX 创建新的 STM32G030F6P6 工程。
2. 初始化 UART，实现 `printf()` 串口输出。
3. 初始化 I²C1，检测 EEPROM 是否返回 ACK。
4. 使用 I²C 向 EEPROM 指定地址写入 1 字节数据。
5. 从相同地址读取 1 字节数据。
6. 比较写入与读取的数据，并通过串口输出实验结果。

**最终目标：**

向 EEPROM 的 `0x10` 地址写入 `0x5A`，随后读取这个地址。若读回的数据为 `0x5A`，串口输出 `EEPROM TEST PASS`。

注意：本任务示例以 24C02 类 EEPROM 为基础。如果实际模块不是 24C02，需要根据实际型号调整设备地址、内部存储地址长度等参数。

---

## 二、第一阶段：创建新的 STM32 工程

### 2.1 创建工程

打开 STM32CubeMX，完成以下操作：

1. 新建 MCU 工程。
2. 搜索并选择 `STM32G030F6P6`。
3. 将工程命名为 `G030_I2C_EEPROM_Test`。
4. 配置调试接口为 `Serial Wire`。
5. 配置系统时钟，可先使用内部 HSI。
6. 配置 UART 和 I²C1 外设。
7. 选择所使用的开发环境，生成工程代码。

### 2.2 UART 配置

根据开发板的实际接线，选择对应的 UART/USART 外设。

推荐参数：

| 参数 | 配置 |
|---|---|
| Mode | Asynchronous |
| Baud Rate | 115200 |
| Word Length | 8 Bits |
| Parity | None |
| Stop Bits | 1 |
| Hardware Flow Control | None |

本阶段仅需要串口发送功能，不需要启用 UART 接收中断。

检查 UART TX 是否连接 USB 转串口模块的 RX，两个设备必须共地。

### 2.3 I²C 配置

在 CubeMX 中启用 I²C1。

| 参数 | 配置 |
|---|---|
| I²C 外设 | I²C1 |
| 工作模式 | I²C |
| 通信速率 | 100 kHz |
| 地址模式 | 7-bit |
| 中断 | 暂不启用 |
| DMA | 暂不启用 |

I²C SCL、SDA 引脚按照实际硬件连接和 CubeMX 引脚复用配置选择。

EEPROM 模块已带有 I²C 上拉电阻，无需另外添加外部上拉电阻。注意确认模块电源电压与 STM32 的 I/O 电平兼容。

### 本阶段验收标准

- 工程创建成功。
- 代码无编译错误。
- MCU 可以正常下载并运行程序。
- 可以在 CubeMX 中找到 UART 和 I²C1 初始化配置。
- 保存好 `.ioc` 工程配置文件。

---

## 三、第二阶段：UART printf 功能测试

### 3.1 实验目标

使用 `printf()` 函数，通过 UART 向电脑串口助手发送文字。

### 3.2 实验要求

1. 在 `main.c` 中添加 `#include <stdio.h>`。
2. 参考之前的 UART 实验，实现 `printf` 重定向。
3. 在 UART 初始化完成之后执行打印。
4. 电脑串口助手设置为 `115200、8N1`。
5. 验证连续输出是否正常。

例如，在主循环中加入：

```c
printf("STM32G030 UART TEST OK\r\n");
HAL_Delay(1000);
```

`printf` 重定向参考（以 Keil 使用 `fputc` 的配置为例，假设选择 USART1）：

```c
int fputc(int ch, FILE *f)
{
    uint8_t data = (uint8_t)ch;

    HAL_UART_Transmit(
        &huart1,
        &data,
        1,
        100
    );

    return ch;
}
```

如果使用的是其他串口，需要修改 `huart1`。如果使用 STM32CubeIDE/GCC，需要根据工程的 C 库配置，选择 `_write()` 等适用的重定向方式。

### 3.3 预期串口输出

```text
STM32G030 UART TEST OK
STM32G030 UART TEST OK
STM32G030 UART TEST OK
```

每隔约 1 秒输出一次。

### 本阶段验收标准

- 能使用 `printf()` 成功输出英文。
- 串口助手无乱码。
- 能正确显示换行。
- 复位 MCU 后，串口输出可以重新开始。

**本阶段通过后，才能进行 EEPROM 的 ACK 测试。**

---

## 四、第三阶段：I²C EEPROM ACK 测试

### 4.1 实验目标

通过 STM32 的硬件 I²C1 检查 EEPROM 是否正常响应设备地址。

### 4.2 基础知识

I²C 从设备具有设备地址。主机发送设备地址后，如果对应的从机存在且准备好通信，它通常会返回 ACK。

例如，对于 A0、A1、A2 全部接地的常见 24C02：

- 7 位设备地址：`0x50`
- STM32 HAL 传入的设备地址：`0x50 << 1`，即 `0xA0`

**注意：ACK 只能证明地址通信正常，不能证明 EEPROM 内部数据读写正确。**

### 4.3 实验要求

学习以下 HAL 函数：

```c
HAL_I2C_IsDeviceReady();
```

参考调用方式：

```c
HAL_I2C_IsDeviceReady(
    &hi2c1,
    0x50 << 1,
    3,
    100
);
```

需要了解：

- `&hi2c1` 的作用是什么？
- 为什么设备地址要左移一位？
- `3` 代表什么？
- `100` 代表什么？
- 函数成功时返回什么？

### 4.4 编程任务

1. 定义 EEPROM 设备地址。
2. 调用 `HAL_I2C_IsDeviceReady()`。
3. 使用 `if/else` 判断返回结果。
4. 如果返回 `HAL_OK`，通过串口输出 `EEPROM ACK OK`。
5. 否则输出 `EEPROM ACK FAIL`。

### 4.5 预期输出

成功时：

```text
STM32G030 BOOT OK
UART PRINTF OK
EEPROM ACK OK
```

失败时：

```text
STM32G030 BOOT OK
UART PRINTF OK
EEPROM ACK FAIL
```

### 本阶段验收标准

- 新工程可以成功检测 EEPROM ACK。
- 能正确判断函数返回值。
- 能通过 `printf()` 报告设备状态。
- EEPROM 不连接或地址错误时，程序能报告检测失败，而不是假装成功。

---

## 五、第四阶段：EEPROM 单字节写入

### 5.1 实验目标

向 EEPROM 的内部存储地址 `0x10` 写入数据 `0x5A`。

### 5.2 学习内容

学习 HAL 函数：

```c
HAL_I2C_Mem_Write();
```

理解参数：

| 参数 | 含义 |
|---|---|
| `hi2c` | I²C 外设句柄 |
| `DevAddress` | EEPROM 设备地址 |
| `MemAddress` | EEPROM 内部存储地址 |
| `MemAddSize` | 内部存储地址长度 |
| `pData` | 写入数据的缓冲区指针 |
| `Size` | 写入数据长度 |
| `Timeout` | 通信超时时间 |

对于 24C02，本实验内部存储地址长度使用 `I2C_MEMADD_SIZE_8BIT`。

### 5.3 编程要求

定义以下变量：

```c
uint8_t write_data = 0x5A;
uint16_t mem_addr = 0x10;
HAL_StatusTypeDef status;
```

使用 `HAL_I2C_Mem_Write()` 完成写入。

写入过程中，需要满足：

1. EEPROM 已通过 ACK 检测。
2. 向内部地址 `0x10` 写入 1 字节。
3. 获取并检查 HAL 函数返回值。
4. 使用 `printf()` 输出写入操作结果。
5. 等待 EEPROM 内部写入完成。

EEPROM 写入后可能需要几毫秒完成内部编程。本阶段可以先使用 `HAL_Delay(10)` 进行等待，并通过 `HAL_I2C_IsDeviceReady()` 再次确认器件就绪。实际写入时间应以 EEPROM 数据手册为准。

### 5.4 预期输出

```text
EEPROM ACK OK
EEPROM WRITE START
WRITE ADDR = 0x10
WRITE DATA = 0x5A
EEPROM WRITE OK
```

如果写入失败，应输出：

```text
EEPROM WRITE FAIL
```

并停止后续正常读回验证，不能继续报告测试成功。

### 本阶段验收标准

- 可以正确调用 EEPROM 写入函数。
- 能正确判断写入是否成功。
- 能打印写入地址和数据。
- 写入操作完成后，器件能恢复就绪状态。

注意检查 EEPROM 的 WP（写保护）引脚是否处于允许写入的状态。

---

## 六、第五阶段：EEPROM 单字节读取及验证

### 6.1 实验目标

从 EEPROM 的内部地址 `0x10` 读取一个字节，并检查是否与写入值一致。

### 6.2 学习内容

学习以下 HAL 函数：

```c
HAL_I2C_Mem_Read();
```

理解它与 `HAL_I2C_Mem_Write()` 的区别。

定义接收变量：

```c
uint8_t read_data = 0;
```

### 6.3 编程任务

1. 确认 EEPROM 写入成功并已就绪。
2. 调用 `HAL_I2C_Mem_Read()`。
3. 从内部存储地址 `0x10` 读取 1 字节。
4. 判断读取函数是否返回 `HAL_OK`。
5. 使用 `printf()` 打印读取数据。
6. 比较 `read_data` 与 `write_data`。
7. 相同则输出 PASS，不同则输出 FAIL。

### 6.4 判断逻辑

参考以下代码：

```c
if (read_data == write_data)
{
    printf("EEPROM TEST PASS\r\n");
}
else
{
    printf("EEPROM TEST FAIL\r\n");
}
```

注意：只有在写入、就绪检查和读取操作均成功的前提下，才可以执行最终数据校验并报告 PASS。

### 6.5 最终预期串口输出

```text
================================
STM32G030 EEPROM TEST
================================
UART PRINTF OK
EEPROM ACK OK

EEPROM WRITE START
WRITE ADDR = 0x10
WRITE DATA = 0x5A
EEPROM WRITE OK

EEPROM READ START
READ ADDR  = 0x10
READ DATA  = 0x5A
EEPROM READ OK

EEPROM TEST PASS
================================
```

### 6.6 附加验证

第一次测试成功之后，完成以下两个测试：

**测试 A：修改写入数据**

将写入数据从 `0x5A` 改为 `0xA5`。

要求读取结果也变为 `0xA5`，说明读写操作确实生效。

**测试 B：断电数据保存**

1. 向 `0x10` 写入 `0x5A`。
2. 关闭电源。
3. 修改程序，使启动后只读取 EEPROM，不执行写入。
4. 重新下载程序并上电。
5. 检查是否还能读取到 `0x5A`。

如果成功，说明 EEPROM 的非易失性存储功能正常。

注意：不要在 `while(1)` 中反复写入 EEPROM。建议每次启动仅执行一次写入测试，之后主循环保持空闲，避免无意义地消耗 EEPROM 写入寿命。

---

## 七、常见问题与排查

| 现象 | 优先排查 |
|---|---|
| UART 无输出 | 串口引脚、波特率、共地、printf 重定向 |
| UART 输出乱码 | 波特率、时钟设置、串口参数 |
| EEPROM 无 ACK | 设备地址、SCL/SDA、供电、上拉电阻 |
| ACK 正常但写入失败 | WP 引脚、写入函数参数、器件忙状态 |
| 写入正常但读取失败 | 内部地址长度、读取地址、写入等待 |
| 读取数据不一致 | 设备型号、地址、写保护、读写操作顺序 |

---

## 八、实验完成标准

只有同时满足以下要求，才认为本次任务完成：

- [ ] 使用 STM32CubeMX 新建独立工程。
- [ ] UART `printf()` 输出正常。
- [ ] I²C EEPROM ACK 检测成功。
- [ ] 成功向 EEPROM 地址 `0x10` 写入 `0x5A`。
- [ ] 成功从 EEPROM 地址 `0x10` 读取数据。
- [ ] 读回数据与写入数据相同。
- [ ] 串口正确输出 `EEPROM TEST PASS`。
- [ ] 修改数据为 `0xA5` 后，验证仍然成功。
- [ ] 完成断电保存验证。

## 九、需要提交的实验成果

1. **完整工程：** 包括 `.ioc` 和全部工程源码。
2. **UART 测试截图：** 串口助手显示 printf 正常。
3. **ACK 测试截图：** 串口助手显示 EEPROM ACK OK。
4. **读写验证截图：** 展示写入地址、写入数据、读取数据和 PASS。
5. **实验总结：** 说明遇到的问题、解决方式和学习收获。

## 十、思考题

实验结束后，尝试独立回答以下问题：

1. I²C 总线为什么需要上拉电阻？
2. ACK 与 NACK 分别代表什么？
3. 为什么 I²C 7 位设备地址要左移一位？
4. EEPROM 的设备地址和内部存储地址有什么区别？
5. `&write_data` 中的 `&` 有什么作用？
6. 为什么 EEPROM 写入后不能立即假定数据已经保存完成？
7. 为什么不能只判断 `HAL_I2C_Mem_Write()` 返回 OK，就直接认定数据一定正确？

**本次实验重点：独立完成外设初始化、HAL 函数调用、返回值判断、串口调试和数据校验。**

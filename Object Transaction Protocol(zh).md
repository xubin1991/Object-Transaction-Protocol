# Object Transaction Protocol (OTP)

为 AI 原生而设计：稳定对象模型 + 确定性事务结构。

**Version:** 1.0 Draft

面向 MCU、BLE、UART、USB 和其他受限链路的开放对象事务协议草案

面向 LLM/Agent 工作流：更容易进行能力发现、任务规划与安全自动化执行。

![alt text](otp_v1(zh).jpg)

## 1. Introduction

### 1.1 Overview

Object Transaction Protocol（OTP）是一种轻量级二进制通信协议。

协议采用 **Object-Oriented Device Model**。

协议不定义具体业务命令，而是将设备抽象为一组可读写的 Object。

协议层仅负责：

- Object Read
- Object Write
- Transaction
- Message Routing

因此协议具有：

- 长期兼容
- 易扩展
- 易于跨 UART / BLE / USB / TCP 复用
- 无业务命令耦合

---

## 2. Design Principles

### 2.1 Device = Object Collection

设备被视为 Object 的集合。上层的 ProtocolVersion、DeviceStatus、Brightness 等概念也都是 Object 的实例。

例如：

```
Device
├── ProtocolVersion (Object)
├── DeviceStatus    (Object)
├── Brightness      (Object)
├── Temperature     (Object, Deprecated since v2.0)
├── ImageBuffer     (Object)
└── Command         (Object)
```

协议只关心：

```
Read Object
Write Object
```

---

### 2.2 Transaction

一次 Payload 可以包含多个 Transaction，这些 Transaction 可以混合包含读操作和写操作。

Transaction 之间互相独立。

---

### 2.3 Object Size Constraint

**所有 Object 的数据大小不能超过 127 字节。**

这是协议的硬性限制，原因如下：

- Transaction Header 中的 BufferLength 字段为 7 位有效值（bit7 保留），最大可表示 127
- 这确保了协议在受限链路（MCU、BLE、UART 等）上的高效性

---

## 3. Byte Order

协议所有多字节整数均采用：

```
Little Endian
```

包括：

- Flags
- MessageID
- Length
- ObjectID
- CRC16
- 所有 Object 数据类型

---

## 4. Frame Format

```
+---------+--------+------+-----------+--------+---------+-------+
| Flags   | Source | Dest | MessageID | Length | Payload | CRC16 |
+---------+--------+------+-----------+--------+---------+-------+
| uint16  | uint8  |uint8 | uint16    | uint16 | N bytes |uint16 |
+---------+--------+------+-----------+--------+---------+-------+
```

Flags 固定为 `0x5AA5`（双字节帧头标识，小端序下链路字节序为 `A5 5A`）。

Source 为发送端地址标识（`0~254`）；Dest 为接收端地址标识（`0~254`），Dest 为 `255`（0xFF）时表示广播。设备收到广播消息后**不应回复**。

Length（N）最大值 1013，即 N ≤ 1013。完整帧长度 = 8（帧头）+ N（Payload）+ 2（CRC16），最大 1023 字节。

CRC16 覆盖 Flags、Source、Dest、MessageID、Length、Payload。CRC16 本身不参与计算。CRC16 计算方法为 CRC-16/MODBUS（多项式 0x8005，初始值 0xFFFF）。

例如读取 ProtocolVersion（ObjectID=0x0000）的请求帧：

```
A5 5A 01 02 00 00 04 00 00 00 00 01 43 F7
│─Flags─│S│D│ MessageID│─Length─│───Payload───│──CRC16──│
```

CRC16 计算结果为 `0xF743`，小端序存放为 `43 F7`。

对应的 Payload 拆解：`00 00` = ObjectID 0x0000（ProtocolVersion），`00` = OffsetAndOp（Read, Offset=0），`01` = BufferLength（读取 1 字节）。

---

## 5. Message ID

MessageID 为 `uint16`。

定义：

```
bit0:     0 = Request, 1 = Response
bit15~1:  Sequence ID
```

解析：

```
seq      = messageID >> 1
response = messageID & 1
```

响应必须保持 Sequence ID 一致。

> **注意:** Sequence ID 可以任意指定，不需要严格递增。其主要用途是区分收到的 ACK 能够与发送的指令匹配。

---

## 6. Payload

Payload 包含一个或多个 Transaction，顺序排列直到 Length 结束。

---

## 7. Transaction

Transaction Header：

```
ObjectID      uint16
OffsetAndOp   uint8
BufferLength  uint8
```

最小 4 Bytes。

OffsetAndOp：

```
bit7:     0 = Read, 1 = Write
bit6~0:   Buffer Offset (0~127)
```

BufferLength 为 7 位有效值（bit7 保留，当前协议永远为 0），取值范围 `1~127`。

Write 包含 Header + Buffer Data。
Read 只有 Header。

Offset 是简单的字节偏移值。单次 Transaction 中 BufferLength 最大值为 127 字节。所有 Object 的 Buffer 大小不能超过 127 字节。
---

## 8. Response Transaction

Response 中的 Transaction 顺序与 Request 严格一一对应，每个 Request Transaction 有且仅有一个 Response Transaction。当收到包含多个 Transaction 的 Payload 时，无论其中个别操作是成功还是失败，设备都必须按顺序为每个 Request Transaction 返回对应的 Response。

### 8.1 Write Response

```
ObjectID      uint16
Status        uint8
```

Status 为 `uint8`：`0x00` Success，其它为 Error Codes。

> **注意:** error codes 应 >= 128，以保持与 read response 一致。

### 8.2 Read Response

```
ObjectID         uint16
StatusOrLength   uint8
Data...
```

规则：

```
0x00~0x7F: 成功，表示 Data 长度
0x80~0xFF: 错误码
```

---

## 9. Object Definition

每个 Object 必须包含以下字段：

| Field            | Description                      |
|------------------|----------------------------------|
| ObjectID         | 唯一标识符                        |
| Name             | 名称                              |
| Type             | 数据类型（见下方）                 |
| Size             | 字节大小（**不能超过 127 字节**）  |
| Access           | 访问权限（见下方）                 |
| State            | 状态（见下方）                     |
| Version          | 首次出现的固件版本                 |
| DeprecatedVersion| 废弃时的固件版本（State 为 Deprecated 时必填） |
| Default          | 默认值（可选）                     |
| Description      | 描述                              |

**Type：**

```
u8, u16, u32, u64, s8, s16, s32, s64, f32, f64, bool, string, byte[]
```

**Size Constraint（大小限制）：**

```
所有 Object 的 Buffer 大小必须不超过 127 字节。
这是由于 Transaction Header 中 BufferLength 字段为 7 位有效值（bit7 保留），
因此单次操作的最大数据长度为 127 字节，Object 总大小也不能超过此限制。
```

Default 字段的表示格式：

- 整数类型（`u8~u64`, `s8~s64`）：直接写数值，如 `100`、`-5`
- 浮点类型（f32, f64）：写小数，如 `3.14`
- bool：写 `true` 或 `false`
- string：写双引号字符串，如 `"default_value"`
- byte[]：写十六进制字符串，如 `0xA5 0x5A`

**Access：**

| Access | Description       |
|--------|-------------------|
| RO     | 只读（状态获取）   |
| WO     | 只写（命令寄存器） |
| RW     | 读写              |

**State：**

| State        | Description                |
|--------------|----------------------------|
| Active       | 活跃                       |
| Deprecated   | 废弃，保持兼容              |
| Reserved     | 保留，不允许访问            |
| Removed      | 移除，ObjectID 不可重新分配 |
| Experimental | 实验性，不保证兼容          |

规则：

- ObjectID 一经发布不得改变语义。
- Deprecated Object 必须保持兼容。
- Removed Object 不可重新分配。
- Reserved Object 不允许访问。
- Experimental Object 不保证兼容。

**ObjectID 空间划分：**

| Range            | Allocation                      |
|------------------|---------------------------------|
| 0x0000~0x00FF    | 协议预定义系统 Object            |
| 0x0100~0x0FFF    | 标准设备 Object                  |
| 0x1000~0xFFFF    | 厂商自定义 Object                |

0x0000 固定为 ProtocolVersion，所有设备必须支持。

---

## 10. Compatibility Rules

协议采用 **Append Only** 原则。

允许：

```
新增 Object
```

禁止：

```
修改已有 Object 含义
```

Deprecated：

```
继续兼容，但不推荐使用
标记废弃时应记录 DeprecatedVersion
```

Version：

```
Object 标记首次出现的固件版本
```

---

## Appendix A: Error Codes

| Code | Error                  | Description              |
|------|------------------------|--------------------------|
| 0x00 | Success                | 成功                     |
| 0x80 | Unknown Object         | 未知 Object              |
| 0x81 | Object Inactive        | Object 未激活            |
| 0x82 | Permission Denied      | 权限不足                 |
| 0x83 | Offset Out Of Range    | 偏移超出范围             |
| 0x84 | Length Out Of Range    | 长度超出范围             |
| 0x85 | Type Mismatch          | 数据类型不匹配           |
| 0x86 | Invalid Value          | 无效的值                 |
| 0x87 | Read Not Supported     | 不支持读取               |
| 0x88 | Write Not Supported    | 不支持写入               |
| 0x89 | Busy                   | 设备忙碌                 |
| 0x8A | Locked                 | 被锁定                   |
| 0x8B | Not Ready              | 设备未就绪               |
| 0x8C | Invalid Sequence       | 无效的序列               |
| 0x8D | Invalid Data           | 无效的数据               |
| 0x8E | CRC Error              | CRC 校验错误             |
| 0x8F | Unsupported Operation  | 不支持的操作             |
| 0x92 | Message Too Large      | 消息过大                 |
| 0x93 | Malformed Payload      | Payload 格式错误         |
| 0x94 | Version Unsupported    | 不支持的协议版本         |
| 0x95 | Address Error          | 地址错误                 |
| 0x96 | Authentication Required| 需要认证                 |
| 0x97 | Authentication Failed  | 认证失败                 |
| 0x98 | Rate Limited           | 请求频率限制             |
| 0x99 | Resource Exhausted     | 资源耗尽                 |
| 0x9A | Internal Error         | 内部错误                 |
| 0x9B | Hardware Failure       | 硬件故障                 |
| 0x9C | Timeout                | 超时                     |
| 0xFF | Unknown Error          | 未知错误                 |

---

## Appendix B: Example Objects

```
Object:       0x0000
Name:         ProtocolVersion
Type:         u16
Size:         2
Access:       RO
State:        Active
Version:      1.0
Description:  OTP 协议版本标识，用于设备发现和兼容性检测
```

```
Object:       0x0100
Name:         DeviceStatus
Type:         u16
Size:         2
Access:       RO
State:        Active
Version:      1.0
Description:  设备当前运行状态，如待机、运行、故障等
```

```
Object:       0x0200
Name:         Brightness
Type:         u8
Size:         1
Access:       RW
State:        Active
Default:      100
Version:      1.0
Description:  显示亮度（0~255），支持读写调节
```

```
Object:       0x0150
Name:         Temperature
Type:         s16
Size:         2
Access:       RO
State:        Deprecated
Version:      1.0
DeprecatedVersion: 2.0
Description:  温度传感器值（已废弃，由新版 Object 替代）
```

```
Object:       0x0300
Name:         Command
Type:         u16
Size:         2
Access:       WO
State:        Active
Version:      1.0
Description:  命令寄存器，写入特定值触发设备操作
```

Command 值定义：

```
1  SaveConfig
2  Reboot
3  FactoryReset
4  ApplyImage
```

```
Object:       0x1000
Name:         ImageBuffer
Type:         byte[]
Size:         120
Access:       RW
State:        Active
Version:      1.0
Description:  图像数据缓冲区，用于固件升级或图片传输
```

---

## Appendix C: Future Extensions

| Extension          | Description                      |
|--------------------|----------------------------------|
| Notification       | 主动通知机制，设备可主动上报状态   |
| Publish / Subscribe| 一对多发布订阅模型               |
| Stream Object      | 流式传输，适用于音视频等大数据    |
| Authentication     | 设备身份认证                     |
| Encryption         | 传输层加密                       |
| Compression        | 传输数据压缩                     |

## License

This project is licensed under the Creative Commons Attribution 4.0
International License (CC BY 4.0). See `LICENSE` for details.

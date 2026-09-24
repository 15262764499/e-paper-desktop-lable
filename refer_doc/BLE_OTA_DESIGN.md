# STM32G070 + CH9140 BLE OTA原理与接口说明

版本：2.0

适用硬件：STM32G070CBT6、CH9140、ST25DV04K、400×300四色墨水屏

传输链路：Android/Windows BLE GATT → CH9140透明串口 → STM32 USART2 115200 8N1

## 1. 设计目标

OTA只更新业务Application，Bootloader永远保留。升级中途掉电会导致Application无效，但重新上电后Bootloader仍会打开CH9140并允许重新上传，从而避免必须拆机连接ST-Link。

CH9140不负责解析或写入STM32固件，它只转发字节。Flash擦除、写入、哈希校验、发布认证和启动跳转全部由STM32 Bootloader完成。

## 2. Flash布局

STM32G070CBT6有128 KiB单Bank Flash，擦除页为2 KiB：

| 区域 | 起始地址 | 结束地址 | 大小 | 页 |
|---|---:|---:|---:|---:|
| Bootloader | `0x08000000` | `0x08005FFF` | 24 KiB | 0..11 |
| Application | `0x08006000` | `0x08017FFF` | 72 KiB | 12..47 |
| 日历底图 | `0x08018000` | `0x0801F7FF` | 30 KiB | 48..62 |
| OTA Metadata | `0x0801F800` | `0x0801FFFF` | 2 KiB | 63 |

Application链接地址和向量表均为`0x08006000`。当前Application约34 KiB，Bootloader约11 KiB，均低于各自上限。

日历底图本体为30,000字节，头部32字节，合计可放入15页。`CalendarFlash_Begin()`只擦除页48..62，不能擦除OTA元数据页63。

## 3. 上电决策

Bootloader位于硬件启动地址`0x08000000`，每次上电或复位首先运行。RTC备份寄存器DR3用于传递一次性OTA请求：

```text
OTA_BOOT_REQUEST_MAGIC = 0x4F544131  // OTA1
```

Bootloader流程：

```text
读取并清除DR3
  ├─ 没有OTA请求且Application有效：立即跳转0x08006000
  ├─ 有OTA请求：打开CH9140并等待升级，最长120秒
  └─ Application无效/升级中断：永久停留在OTA恢复模式
```

为支持首次工厂迁移，OTA元数据页完全擦除且Application向量合法时允许启动一次由SWD写入的重定位Application。正式OTA在擦除Application前必定先写入`OTAP`事务标志，因此中断升级不会错误走工厂回退路径。

## 4. Application进入OTA

### 4.1 OTA_QUERY

完成`HELLO/AUTH_PROVE`后发送空payload：

```text
type = 0x20
```

返回`OTA_INFO (0x85)`，payload固定23字节：

| 偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 1 | result |
| 1 | 4 | Application版本LE32 |
| 5 | 2 | Bootloader协议版本LE16 |
| 7 | 4 | Application最大字节数，当前73728 |
| 11 | 4 | Application基地址，`0x08006000` |
| 15 | 2 | STM32设备ID，当前`0x0460` |
| 17 | 2 | 硬件版本，当前`1` |
| 19 | 4 | 产品型号，当前`0x00000001` |

### 4.2 OTA_ENTER

payload固定24字节：

```text
firmwareVersionLE32 | nonceLE32 | otaEnterTag(16)
```

```text
otaEnterTag = first16(HMAC-SHA256(
  sessionKey,
  ASCII("EPD-OTA-ENTER-V1") || sequenceLE16 ||
  firmwareVersionLE32 || nonceLE32
))
```

验证成功后Application关闭墨水屏电源，在RTC DR3写入OTA请求，发送成功ACK并执行`NVIC_SystemReset()`。

## 5. BLE断开与重连

板上Q8由PB10控制且低电平使能。MCU复位时PB10暂时高阻，由外部电阻把Q8关闭，因此CH9140会短暂掉电，原BLE连接必然可能断开。

Bootloader重新配置：

- PB10=LOW：打开3V3_BT；
- PB15=HIGH：CH9140正常工作；
- PA2=USART2_TX，PA3=USART2_RX；
- USART2=115200、8N1；
- 上电稳定等待约520 ms。

客户端必须把OTA_ENTER后的首次断开视为正常状态，并重新扫描、连接、发现FFF0服务、订阅FFF1、向FFF2写入。

## 6. Bootloader数据包

仍使用普通协议的包封装：

```text
45 50 | version=01 | type | sequenceLE16 | payloadLengthLE16 |
payload | IEEE-CRC32LE
```

CRC32覆盖`version`至payload，不覆盖Magic。payload最大132字节。BLE ATT分包边界不等于应用数据包边界，客户端和Bootloader都按字节流解析。

### 6.1 消息类型

| 类型 | 值 | 方向 | 说明 |
|---|---:|---|---|
| BOOT_HELLO | `0x30` | 客户端→Boot | 空payload |
| OTA_BEGIN | `0x31` | 客户端→Boot | 80字节发布清单 |
| OTA_DATA | `0x32` | 客户端→Boot | offset+最多128字节 |
| OTA_END | `0x33` | 客户端→Boot | 空payload |
| OTA_ABORT | `0x34` | 客户端→Boot | 空payload，保持Application无效 |
| PING | `0x14` | 客户端→Boot | 链路检查 |
| BOOT_HELLO_RSP | `0xB0` | Boot→客户端 | Bootloader身份和能力 |
| OTA_ACK | `0xB1` | Boot→客户端 | 命令结果和下一offset |
| OTA_STATUS | `0xB2` | Boot→客户端 | 阶段与进度 |

### 6.2 BOOT_HELLO_RSP

payload固定37字节：

| 偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 1 | result |
| 1 | 1 | mode=`1`，表示Bootloader |
| 2 | 2 | Bootloader协议版本 |
| 4 | 1 | Boot状态 |
| 5 | 12 | board ID |
| 17 | 4 | 当前有效固件版本，无则0 |
| 21 | 4 | 最大Application大小 |
| 25 | 4 | 当前期待offset |
| 29 | 2 | STM32设备ID，当前`0x0460` |
| 31 | 2 | 硬件版本，当前`1` |
| 33 | 4 | 产品型号，当前`0x00000001` |

### 6.3 OTA_BEGIN

payload固定80字节，可直接从`.epota`头部偏移8开始读取：

| 偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 2 | STM32设备ID=`0x0460` |
| 2 | 2 | 硬件版本=`1` |
| 4 | 4 | 产品型号=`0x00000001` |
| 8 | 4 | firmwareVersion LE32 |
| 12 | 4 | firmwareSize LE32 |
| 16 | 32 | firmware SHA-256 |
| 48 | 32 | releaseTag HMAC-SHA256 |

Bootloader在擦除旧Application之前完成STM32设备ID、硬件版本、产品型号、固件版本、大小和releaseTag验证。升级包不包含STM32唯一UID，因此同一硬件型号的所有设备共用一个文件。当前版本要求新固件版本严格大于已验证版本。

验证通过后先擦除元数据页并写`OTAP`事务标志，再擦除全部Application页。擦除阶段先发STATUS，擦除结束后才返回OTA_BEGIN成功ACK。

### 6.4 OTA_DATA

```text
offsetLE32 | data(1..128)
```

- offset从0严格连续增长；
- 每块写入Flash并回读成功后才ACK；
- ACK丢失时可重发已经完整提交的旧块；
- 最后一块可以不是8字节倍数，Bootloader用`0xFF`补齐Double Word，但SHA只覆盖真实固件长度；
- OTA_DATA 15秒无活动进入ERROR，重新发送OTA_BEGIN可从头恢复。

### 6.5 OTA_END

payload为空。Bootloader依次执行：

1. 补齐并写入最后一个Double Word；
2. 结束流式SHA-256；
3. 从Flash重新读取Application并再次计算SHA-256；
4. 两个结果都与清单hash比较；
5. 检查初始MSP位于`0x20000000..0x20009000`；
6. 检查Reset Handler为Thumb地址且位于固件长度内；
7. 写入固件版本、长度、hash和releaseTag；
8. 最后一个Flash写操作提交`OTAV`有效标志；
9. 返回完成状态并复位。

### 6.6 ACK与STATUS

Bootloader ACK固定7字节，与普通图片ACK字段顺序不同：

```text
ackedType(u8) | result(u8) | bootState(u8) | nextOffset(u32 LE)
```

STATUS与普通协议结构相同：

```text
bootState(u8) | result(u8) | progressPermille(u16 LE) |
received(u32 LE) | expected(u32 LE)
```

Boot状态：0=IDLE，1=ERASING，2=RECEIVING，3=VERIFYING，4=COMPLETE，5=ERROR。

Boot result：0=OK，1=包损坏，2=协议版本错误，3=设备不匹配，4=状态错误，5=长度错误，6=offset错误，7=hash错误，8=发布认证失败，9=Flash错误，10=超时，11=拒绝降级，12=向量表错误。

## 7. EPO2通用升级包

使用命令生成：

```powershell
python tools/package_ota.py build/epaper_project.bin `
  --version 1.0.2 `
  --development-key
```

输出扩展名为`.epota`。文件头固定88字节：

| 文件偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 4 | ASCII `EPO2` |
| 4 | 1 | 容器版本=`2` |
| 5 | 1 | flags，当前=`0` |
| 6 | 2 | 文件头长度=`88` |
| 8 | 2 | STM32设备ID=`0x0460` |
| 10 | 2 | 硬件版本=`1` |
| 12 | 4 | 产品型号=`0x00000001` |
| 16 | 4 | 固件版本 |
| 20 | 4 | 固件长度 |
| 24 | 32 | SHA-256 |
| 56 | 32 | releaseTag |
| 88 | N | Application二进制 |

Android发送OTA_BEGIN时使用文件偏移8..87，发送OTA_DATA时使用偏移88之后的数据。

`releaseTag`计算方式：

```text
HMAC-SHA256(
  releaseKey,
  ASCII("EPD-OTA-RELEASE-V2") || stm32DeviceIdLE16 ||
  hardwareRevisionLE16 || productIdLE32 ||
  firmwareVersionLE32 || firmwareSizeLE32 || firmwareSHA256
)
```

仓库中的key仅用于开发测试。安卓只收到已打包的`.epota`文件，不得获得releaseKey。除非硬件真的发生变化，否则不要通过命令行覆盖`--product-id`或`--hardware-revision`。

## 8. 安全边界

当前V2实现包含两层认证：

1. NFC派生session key保护OTA_ENTER，防止未配对手机让正常设备进入升级模式；
2. Bootloader独立release key验证升级包，持有NFC pairing key的客户端也不能制作任意固件。

board ID仍用于NFC配对、`HELLO`设备匹配以及`OTA_ENTER`会话认证，但不进入EPO2发布签名。这样既保持每台设备的近场授权，又允许同型号设备共用一个正式升级包。

开发版发布认证使用HMAC-SHA256。其局限是Bootloader内部仍保存对称密钥；如果攻击者能读出Flash，就能伪造固件。量产建议替换为ECDSA P-256或Ed25519，Bootloader只保存公钥，私钥留在离线发布系统或HSM中。

在硬件实测稳定以后，应通过STM32CubeProgrammer Option Bytes把Bootloader页0..11设置为WRP写保护。不要在尚未验证恢复路径时直接修改RDP/WRP；错误Option Bytes可能造成调试和返修困难。

OTA不提供固件内容加密，固件二进制可能被观察，但不能在不知道发布密钥的情况下修改并通过认证。

## 9. 断电恢复

本芯片剩余空间不足以保留完整A/B双应用，因此OTA开始后旧Application会被擦除。安全性来自不可擦除的Bootloader和事务标志：

- 擦除Application前写入`OTAP`；
- 任何阶段断电后都不会跳转到部分固件；
- 上电后Bootloader重新打开BLE并永久等待有效OTA_BEGIN；
- 只有完整hash、回读hash、发布认证和向量表全部通过后才写`OTAV`；
- `OTAV`写入成功后复位并启动新Application。

## 10. 首次烧录与构建

构建Application：

```powershell
make
```

构建Bootloader：

```powershell
make -f Bootloader/Makefile
```

首次必须使用ST-Link或串口ROM Bootloader分别烧录：

1. `build_bootloader/epaper_bootloader.hex`，地址由HEX指定为`0x08000000`；
2. `build/epaper_project.hex`，地址由HEX指定为`0x08006000`。

首次安装可全片擦除；以后单独更新Application时不能全片擦除，否则会删除Bootloader。烧录后先确认正常上电能够从Bootloader跳转到Application，再启用页0..11的WRP。

每次发布必须递增`OTA_APPLICATION_VERSION`并用相同版本生成`.epota`。客户端不能只修改容器版本字段，因为releaseTag会失效。

从旧的设备专用EPO1/Bootloader协议V1迁移到EPO2时，必须通过ST-Link更新Bootloader，因为Bootloader本身不能通过本协议自更新。建议全片擦除后依次烧录新版Bootloader和Application；如果必须保留其他Flash数据，至少还要擦除旧OTA元数据页`0x0801F800..0x0801FFFF`。旧Bootloader不能接收EPO2，旧EPO1包也不能用于新版Bootloader。

## 11. 验收清单

- Bootloader HEX最低地址为`0x08000000`且不超过`0x08005FFF`；
- Application HEX最低地址为`0x08006000`且不超过`0x08017FFF`；
- 日历写入不触碰页63；
- 普通启动不会出现明显BLE额外等待；
- OTA_ENTER失败不会复位；
- OTA_ENTER成功后客户端能处理一次预期断连；
- 修改清单或固件任一字节均不能启动；
- 传输任意阶段断电后仍能重新发现CH9140并完整恢复；
- OTA完成后版本查询、图片传输、RTC、日历和5分钟温湿度刷新全部正常。

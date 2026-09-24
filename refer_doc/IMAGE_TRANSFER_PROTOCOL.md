# NFC + BLE 墨水屏图片传输协议（v1）

## 1. 硬件链路

- NFC：ST25DV04K。手机只通过 NFC 取得目标设备 ID、协议版本和配对密钥。
- BLE：CH9140 透明串口，STM32 USART2 为 115200、8N1。
- BLE 服务：`FFF0`；手机订阅 `FFF1` Notify 接收 STM32 数据，向 `FFF2` Write 写入 STM32 数据。
- 屏幕：400×300、原生 2bpp，固定一帧 30,000 字节。
- 普通照片不写 Flash，按顺序直接写入屏控制器 RAM。日历模式会把一份已认证的 30,000 字节基础模板保存在 Flash 的 30 KiB 日历区，用于本地跨日重绘；最后 2 KiB 页面保留给 OTA 元数据。

## 2. NFC NDEF

NDEF 使用 MIME 类型 `application/vnd.epaper.pair`，payload 为：

| 偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 4 | ASCII `EPD1` |
| 4 | 1 | 协议版本，当前为 `1` |
| 5 | 12 | STM32 96-bit UID，小端字节顺序，作为 board ID |
| 17 | 16 | 由固件根密钥和 STM32 UID 派生的本设备配对密钥 |
| 33 | 1 | flags；bit0=必须执行 HMAC 验证 |

安卓应用应声明该 MIME 的 NFC intent-filter。读取成功后保存 board ID 和密钥，再连接附近的 CH9140。此方案把“能够近距离读取 NFC”作为首次配对凭据；当前固件通过 UID 自动派生每台设备的不同密钥。量产时仍应保护根密钥，安全要求更高时应改用安全元件。

## 3. BLE 串口数据包

所有多字节整数均为小端：

| 字段 | 长度 | 说明 |
|---|---:|---|
| Magic | 2 | `45 50`（ASCII `EP`） |
| Version | 1 | `01` |
| Type | 1 | 消息类型 |
| Sequence | 2 | 安卓递增序号；响应复用请求序号 |
| Payload length | 2 | payload 字节数 |
| Payload | N | 最大 132 字节 |
| Packet CRC32 | 4 | IEEE CRC32，覆盖 Version 至 Payload，不覆盖 Magic |

CRC32 参数：poly `0xEDB88320`、初值 `0xFFFFFFFF`、最终异或 `0xFFFFFFFF`。

BLE 写入可能把一个包拆成多个 ATT Write，也可能把多个包连续送到串口；STM32 解析器不依赖 BLE 分包边界。安卓应按 `MTU - 3` 拆分写入，且每个图片块等待 ACK 后再发下一块。

## 4. 认证

### 4.1 HELLO

安卓发送 `HELLO_REQ (0x01)`，payload 是 NFC 取得的 12-byte board ID。

STM32 返回 `HELLO_RSP (0x81)`：

| 偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 1 | result |
| 1 | 12 | board ID |
| 13 | 16 | 本次随机 challenge |
| 29 | 2 | width = 400 |
| 31 | 2 | height = 300 |
| 33 | 4 | frame bytes = 30000 |
| 37 | 1 | format = 1（native 2bpp） |
| 38 | 2 | max image data = 128 |
| 40 | 1 | 当前状态 |
| 41 | 2 | capabilities：bit0=日历模板、bit1=TIME_SYNC、bit2=本地RTC日历、bit3=持久模板、bit4=BLE OTA、bit5=三模式灯效控制 |

### 4.2 AUTH_PROVE

安卓计算：

```text
authProof = first16(HMAC-SHA256(
  pairingKey,
  ASCII("EPD-AUTH-V1") || boardId || challenge
))
```

然后发送 `AUTH_PROVE (0x02)`，payload 为 16-byte `authProof`。STM32 常量时间校验成功后，只授权一张图片；完成、取消、错误或超时都会清除授权。

双方同时计算本次会话密钥：

```text
sessionKey = HMAC-SHA256(
  pairingKey,
  ASCII("EPD-SESSION-V1") || boardId || challenge
)
```

### 4.3 TIME_SYNC (0x04)

仅允许在认证成功后发送。payload 固定 26 字节：

```text
epochSeconds(u64 LE) | utcOffsetMinutes(i16 LE) | timeTag(16)
```

```text
timeTag = first16(HMAC-SHA256(
  sessionKey,
  ASCII("EPD-TIME-V1") || sequenceLE16 || epochSecondsLE64 ||
  utcOffsetMinutesLE16
))
```

固件接受的时间范围为 2000-01-01 至 2099-12-31，时区偏移范围为 -840 至 +840 分钟。验证成功后 RTC 保存本地时间。

## 5. 图片传输

### 5.1 IMAGE_BEGIN (0x10)

payload 固定 18 字节：

| 偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 4 | image ID，安卓生成 |
| 4 | 2 | width = 400 |
| 6 | 2 | height = 300 |
| 8 | 1 | format = 1 |
| 9 | 1 | renderProfile：`0x00`=普通照片，`0x11`=日历布局 V1 |
| 10 | 4 | size = 30000 |
| 14 | 4 | 整帧 IEEE CRC32 |

STM32 先返回 `PREPARING` 状态，随后才给墨水屏上电、复位并初始化。安卓必须等收到成功 ACK 后再发图片块。

日历模式必须先完成 `TIME_SYNC`。该模式在 PREPARING 阶段擦除保留模板区，在所有数据、CRC 与 HMAC 校验成功后才标记模板有效，然后逐行叠加日期、AHT20 室内温度及今日红框并刷新屏幕。

### 5.2 IMAGE_DATA (0x11)

payload 为 `offset(u32 LE) + data(1..128 bytes)`。数据必须从 offset 0 严格连续发送。STM32 每块写入屏控制器后返回 ACK，其中包含下一期待 offset。重复发送一个已经完整接收的旧块只会重发 ACK，不会重复写屏。

### 5.3 IMAGE_END (0x12)

为图片内容生成认证标签：

```text
imageTag = first16(HMAC-SHA256(
  sessionKey,
  ASCII("EPD-IMAGE-V1") ||
  imageIdLE32 || sizeLE32 || imageCrcLE32 || frameBytes
))
```

payload 固定 24 字节：`imageId(u32) + imageCrc32(u32) + imageTag(16)`。

日历模式使用独立标签域，且把渲染配置纳入认证：

```text
calendarTag = first16(HMAC-SHA256(
  sessionKey,
  ASCII("EPD-CALENDAR-V1") || imageIdLE32 || sizeLE32 ||
  imageCrcLE32 || renderProfile || frameBytes
))
```

STM32 只有在字节数、CRC32 和 HMAC 全部正确时才发送刷新命令。刷新期间返回 `REFRESHING`；完成后关屏电源、清除会话并返回 `COMPLETE`。安卓端刷新等待超时建议至少 70 秒。

### 5.4 CANCEL / PING

- `CANCEL (0x13)`：中止传输、丢弃会话、拉低屏幕复位和电源，不刷新不完整图片。
- `PING (0x14)`：返回 ACK，用于检查透明串口链路。

### 5.5 OTA_QUERY / OTA_ENTER

- `OTA_QUERY (0x20)`：认证后发送空 payload，固件返回23字节的 `OTA_INFO (0x85)`，包含当前应用版本、Bootloader协议版本、最大应用长度、应用基地址、STM32设备ID、硬件版本和产品型号。
- `OTA_ENTER (0x21)`：认证后请求进入 Bootloader。payload 和 HMAC 规则见 [BLE_OTA_DESIGN.md](BLE_OTA_DESIGN.md)。成功 ACK 后 STM32 复位，CH9140 会短暂断电，安卓必须重新扫描和连接。

OTA固件块、EPO2通用升级清单和Bootloader响应不使用本节的图片状态机。升级包不绑定唯一board ID，同型号设备共用一个文件；完整定义以 [BLE_OTA_DESIGN.md](BLE_OTA_DESIGN.md) 为准。

## 6. 响应

`ACK (0x82)` payload：

```text
ackedType(u8) | result(u8) | nextOffset(u32 LE) | state(u8)
```

`STATUS (0x83)` payload：

```text
state(u8) | result(u8) | progressPermille(u16 LE) |
received(u32 LE) | expected(u32 LE)
```

`state` 数值：0=IDLE，1=AUTHENTICATED，2=PREPARING，3=RECEIVING，4=VERIFYING，5=REFRESHING，6=COMPLETE，7=ERROR。

主要 result：0=OK，3=设备不匹配，4=未授权，5=忙，6=尺寸错误，7=偏移错误，8=CRC错误，9=墨水屏错误，10=超时，11=串口溢出，12=HMAC验证失败，13=传感器错误，14=时间错误，15=Flash错误，16=OTA请求被拒绝。

## 7. 安卓图片处理要求

1. 使用系统 Photo Picker 选择图片。
2. 按 4:3 比例 center-crop，使图片铺满，不留白边；缩放到 400×300。
3. 将每个像素量化成四种颜色：黑=0、白=1、黄=2、红=3。可选误差扩散抖动以改善照片效果。
4. 行优先、从左上角开始打包；每字节 4 像素，第一个像素放 bits 7..6，随后为 bits 5..4、3..2、1..0。
5. 得到恰好 30,000 字节后计算 CRC32、会话 HMAC，再按上述协议发送。

墨水屏本身断电后可能继续保持最后画面，这是其物理特性；但 STM32 不保存原始图片，复位后无法重新发送上一张图。


## 2026-09-14 灯效扩展（v1 兼容）

新增 `LED_SET=0x05`、`LED_GET=0x06`、`LED_STATE=0x86`，能力位 `0x0020`。
认证后可选择 0=关闭、1=湿度呼吸、2=中心展开、全亮保持 1 秒后渐暗的循环扩散呼吸。NFC 临时覆盖不属于可设置模式。
SET payload 为 mode+16 字节 HMAC，GET 为空；成功回复 35 字节 LED_STATE，错误走既有 ACK。
完整偏移表、HMAC 域、状态同步和示例动画公式见 [ANDROID_LED_CONTROL.md](ANDROID_LED_CONTROL.md)。

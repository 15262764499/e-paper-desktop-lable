# 2026-09-11 硬件适配与呼吸灯测试

> 以下为 2026-09-11 历史硬件演示记录。当前三模式驱动、接口和时序以 [LED_LOGIC.md](LED_LOGIC.md) 与 [ANDROID_LED_CONTROL.md](ANDROID_LED_CONTROL.md) 为准；旧 SetMask/共用 OE PWM 演示已替换。

固件按 `SCH_Schematic3_2026-09-11.pdf` 接线适配。

| MCU 引脚 | 功能 | 74HC595 引脚 |
| --- | --- | --- |
| PA4 / AF1 | SPI2_MOSI | DS，14 脚 |
| PA0 / AF0 | SPI2_SCK | SHCP，11 脚 |
| PB14 / GPIO | 锁存脉冲 | STCP，12 脚 |
| PA8 / AF2 | TIM1_CH1，低有效 PWM | OE#，13 脚 |

SPI2 使用模式 0、MSB 优先、8 位、单线发送；当前 16 MHz 时钟下速率为 500 kbit/s。引脚复用依据 [ST 数据手册表 12](https://www.st.com/resource/en/datasheet/stm32g070cb.pdf)。MR# 由图中的 R70 上拉。

图中实际是 Q1～Q7 接 7 颗 LED，Q0（15 脚）未连接；Q7S 是级联输出，不是第八路灯。LED 阳极通过 1 kΩ 接 3V3_SYS，阴极接 Q 输出，因此输出低电平点亮。初始化移入 `0x01`，让 Q1～Q7 有效、Q0 关闭。

上电进入循环同步呼吸：1.5 秒渐亮、1.5 秒渐暗。PWM 为 1 kHz，亮度每 10 ms 更新，使用平方曲线改善暗部变化。SysTick 只计算亮度并写 CCR1，不进行 SPI 传输或延时；正常中断开启时，墨水屏阻塞等待不会停止呼吸。Flash 擦写或关闭中断仍可能使亮度更新短暂停顿。

`Core/Inc/led_595.h` 中的 `LED595_BREATH_PERIOD_MS` 可修改周期，`LED595_DEMO_MASK` 可选择输出。掩码的 bit0～bit7 对应 Q0～Q7，1 表示选中；若以后补接 Q0 的 LED，可改为 `0xFF`。`LED595_SetMask()` 应从主循环调用，支持后续流水灯效果；当前演示为同步呼吸。

NFC、BLE、AHT20 与 I2C 上拉均按直供电处理，代码不再配置 PB9、PB10、PB5 为电源使能。NFC 直接探测 I2C 地址，不再切换电源极性。BLE 使用 PB15：`Bluetooth_Sleep()` 拉低 SLP，`Bluetooth_Wake()` 拉高并等待 500 ms。休眠前需结束 BLE 收发；当前传图程序默认唤醒，不自动空闲休眠，以便手机继续连接。独立墨水屏诊断演示中的原断电操作已改为 SLP 休眠。Bootloader 在 OTA 期间保持 BLE 唤醒，并把 PA8 拉高关闭灯输出。

构建：`make -j4 firmware`。

本次应用和 Bootloader 已编译、链接成功，Flash 的 text+data 分别为 36224 B / 72 KiB 和 11160 B / 24 KiB。链接器报告了 newlib 的 `_read/_write/_close/_lseek` 占位实现以及 RWX LOAD 段警告；本次未调整系统调用或链接脚本。尚未烧录或进行实物测试。

- 应用：`build/epaper_project.hex`；BIN 加载地址为 `0x08006000`。
- Bootloader：`build_bootloader/epaper_bootloader.hex`；BIN 加载地址为 `0x08000000`。
- HEX 自带地址。应用依赖本项目 Bootloader，不应将应用 BIN 写到 Flash 起始地址。

`files/墨水屏项目.ioc` 已同步删除旧电源引脚并加入 SPI2/TIM1/PB14 配置。呼吸灯逻辑和初始化由 `led_595.c` 管理；本次没有运行 CubeMX 重新生成工程。以后重新生成时需保留 `LED595_Init()`、SysTick 中的 `LED595_Tick()` 和 Makefile 源文件项。

板上检查：上电观察 7 颗灯同步呼吸；示波器检查 PA0 的 8 个时钟及其后的 PB14 锁存上升沿，PA8 应为低有效 1 kHz PWM。随后验证手机连接、NFC 配对、传图和 OTA。编译与静态检查无法代替上述实物验证。

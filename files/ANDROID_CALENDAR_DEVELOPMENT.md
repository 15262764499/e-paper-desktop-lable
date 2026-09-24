# 安卓端日历功能开发说明

版本：0.1（联调草案）  
目标硬件：STM32G070CBT6 + CH9140 + ST25DV04K + AHT20 + GDEM042F86 400×300 四色墨水屏  
依赖协议：[IMAGE_TRANSFER_PROTOCOL.md](IMAGE_TRANSFER_PROTOCOL.md)

## 1. 目标

安卓应用在用户触碰 NFC 后完成以下流程：

1. 读取 NFC 配对信息并连接 CH9140 BLE。
2. 与 STM32 完成 `HELLO/AUTH` 身份认证。
3. 接收 STM32 上传的 AHT20 室内温湿度。
4. 使用手机时间校准 STM32 RTC。
5. 读取 Android 系统日历中今天的日程事件。
6. 申请前台定位权限，获取手机当前位置，并通过项目默认天气提供商 Open-Meteo 获取室外天气。
7. 绘制 400×300 日历基础画面，包括日历框、星期标题、天气和今日日程。
8. 将基础画面量化、打包并通过现有安全图片协议发送。
9. STM32 在基础画面上叠加日期数字、当前日期、室内温度和今日高亮框，再刷新墨水屏。
10. 手机不再连接时，STM32 依靠 RTC 在跨日后重新绘制日期、室内温度和高亮框。

本文中的“今日任务”默认指 Android `Calendar Provider` 中的日历事件，不包括 Google Tasks、Microsoft To Do 等独立待办服务。若产品需要真正的待办事项，需要另外接入对应账号和网络 API。

## 2. 两端职责

| 内容 | 安卓端 | STM32端 |
|---|---|---|
| 400×300基础画面 | 生成并发送 | 接收并保存日历模板 |
| 固定日历框 | 绘制 | 不绘制 |
| 星期标题 | 绘制 | 不绘制 |
| 室外天气和图标 | 获取、绘制 | 不解析天气 |
| 今日系统日历事件 | 读取、排版、绘制 | 不保存事件文本 |
| 月份日期数字1～31 | 预览时模拟，传输画面中留白 | 根据RTC绘制 |
| 当前年月日 | 预览时模拟，传输画面中留白 | 根据RTC绘制 |
| 今日高亮框 | 预览时模拟，传输画面中留白 | 根据RTC绘制 |
| 室内温度 | 用于最终预览 | 从AHT20读取并绘制 |
| 时间维护和跨日 | 发送校时数据 | RTC维护并在零点刷新 |

安卓发送的是“基础模板”，不是最终画面。安卓预览必须在基础模板的副本上模拟 STM32 叠加层，不能把叠加层写进实际发送的30,000字节，否则会出现日期和高亮重复。

## 3. 必须解决的模板保存问题

当前固件把图片直接写入墨水屏控制器RAM，不在STM32中保存。屏幕断电后，STM32第二天无法重新得到安卓绘制的天气和任务底图。

日历模式需要嵌入式端新增一个保留Flash区域：

- 建议保留最后32KB内部Flash。
- 保存30,000字节安卓日历基础模板及少量元数据。
- 普通照片仍然不保存。
- 模板只有在长度、CRC32和HMAC全部验证成功后才标记为有效。
- 收到相同CRC的模板时不重复擦写Flash。
- 固件升级可以擦除模板；升级后等待用户再次碰NFC生成即可。

当前固件约使用25KB Flash，STM32总Flash为128KB，预留32KB模板区后仍有明显余量。链接脚本必须缩小应用Flash长度，避免程序覆盖模板区。

如果不增加模板存储，STM32只能在手机连接当天叠加字符，无法在第二天保留安卓绘制的天气和任务区域后再进行完整刷新。

## 4. 固定画布和布局版本

所有绘图使用固定的400×300逻辑像素，不使用Android的`dp`、屏幕密度或自动缩放。

布局版本：`CALENDAR_LAYOUT_V1 = 1`

### 4.1 区域定义

| 区域 | 坐标 | 所有者 | 内容 |
|---|---|---|---|
| 日期标题 | `x=0..175, y=0..47` | STM32 | `2026年8月20日`等 |
| 室内温度 | `x=176..279, y=0..47` | STM32 | `室内 25.6℃` |
| 星期标题 | `x=0..279, y=48..71` | 安卓 | 一、二、三、四、五、六、日 |
| 月历网格 | `x=0..279, y=72..299` | 安卓+STM32 | 安卓画线；STM32画日期和高亮 |
| 室外天气 | `x=280..399, y=0..115` | 安卓 | 天气、室外温度、天气图标 |
| 今日日程 | `x=280..399, y=116..299` | 安卓 | 系统日历事件 |

屏幕右侧和左侧之间在 `x=279` 处绘制黑色分隔线。

### 4.2 月历网格

- 月历宽度280像素，每列40像素。
- 星期标题高度24像素。
- 日期区固定6行，每行38像素。
- 竖线位置：`0, 40, 80, 120, 160, 200, 240, 279`。
- 横线位置：`48, 72, 110, 148, 186, 224, 262, 299`。
- STM32根据当月1日星期和当月天数确定每个日期所在单元格。
- STM32将日期数字放在单元格左上角，建议偏移 `(4, 3)`。
- 今日高亮框在单元格边界内缩2像素，建议使用红色2像素描边。
- 周六、周日日期可以使用红色，工作日使用黑色。

### 4.3 安卓必须留白的区域

基础模板中以下位置必须保持纯白色 `#FFFFFF`，不能绘制文字、图标或抖动噪声：

- 日期标题区域内部。
- 室内温度区域内部。
- 所有日期单元格内部，网格线除外。
- 今日高亮框可能经过的单元格内缩0～4像素区域。

否则STM32叠加字符时无法可靠清除旧日期和旧高亮框。

## 5. NFC、BLE和传输流程

```text
NFC碰一碰
  -> BLE扫描和连接
  -> 订阅FFF1
  -> HELLO_REQ / HELLO_RSP
  -> AUTH_PROVE
  -> SENSOR_RSP（室内温湿度）
  -> TIME_SYNC（新增）
  -> 查询今天的系统日历
  -> 获取或读取缓存天气
  -> 生成日历基础模板和最终预览
  -> IMAGE_BEGIN（日历模式）
  -> IMAGE_DATA
  -> IMAGE_END
  -> 等待COMPLETE
```

传输仍沿用现有分包、CRC32、ACK、重试和图片HMAC规则。日历模板仍然必须恰好是30,000字节。

## 6. 协议扩展要求

本节需要安卓和固件同步实现；当前固件尚未支持这些扩展。

### 6.1 能力声明

建议在`HELLO_RSP`末尾增加两字节`capabilities`：

```text
bit0 = 支持日历基础模板
bit1 = 支持TIME_SYNC
bit2 = 支持RTC自动跨日刷新
bit3 = 支持内部Flash模板保存
```

安卓只有在必要能力全部存在时才显示“发送日历”。旧固件仍可继续使用普通照片模式。

### 6.2 TIME_SYNC

建议新增：

```text
TIME_SYNC = 0x04
```

payload：

| 偏移 | 长度 | 内容 |
|---:|---:|---|
| 0 | 8 | Unix epoch seconds，u64小端 |
| 8 | 2 | 当前UTC偏移分钟数，i16小端；中国为480 |
| 10 | 16 | timeTag |

认证标签：

```text
timeTag = first16(HMAC-SHA256(
  sessionKey,
  ASCII("EPD-TIME-V1") ||
  sequenceLE16 || epochSecondsLE64 || utcOffsetMinutesLE16
))
```

STM32验证成功后设置RTC并返回普通`ACK`。安卓应在生成日历前发送校时，设备拒绝校时时不得发送日历模板。

### 6.3 日历渲染配置

现有`IMAGE_BEGIN`偏移9的`reserved`改为`renderProfile`：

```text
0x00 = 普通图片，不叠加、不保存
0x11 = 日历布局V1，叠加日期/室内温度/高亮并保存模板
```

普通图片继续使用原有`EPD-IMAGE-V1`标签。日历模式使用独立域，防止渲染配置被未授权修改：

```text
calendarTag = first16(HMAC-SHA256(
  sessionKey,
  ASCII("EPD-CALENDAR-V1") ||
  imageIdLE32 || sizeLE32 || imageCrcLE32 ||
  renderProfile || frameBytes
))
```

`IMAGE_END`结构可以保持不变，但日历模式下最后16字节应填`calendarTag`。

## 7. AHT20室内温湿度

当前协议已经定义：

```text
SENSOR_REQ = 0x03
SENSOR_RSP = 0x84
```

认证成功后STM32会主动发送一次`SENSOR_RSP`。安卓需要验证传感器HMAC后再使用数据。

应用数据模型：

```kotlin
data class IndoorReading(
    val temperatureCentiC: Int,
    val humidityCentiPercent: UInt,
    val receivedAt: Instant,
    val valid: Boolean,
)
```

显示格式：

- 室内温度保留一位小数，例如`25.6℃`。
- 湿度本版本不由STM32叠加，可以保留在安卓预览或以后扩展。
- `SENSOR_RSP`失败时，最终预览显示`室内 --.-℃`。
- STM32本地跨日刷新时重新读取AHT20，使用最新温度，不依赖安卓预览值。
- 日历模式下STM32每5分钟重新读取一次AHT20，并在顶部同时显示室内温度和湿度；该周期刷新不会在普通照片模式下执行。

## 8. 读取Android系统日历

Android官方`Calendar Provider`通过`CalendarContract`提供日历事件。读取需要危险权限`READ_CALENDAR`；只读取事件不需要`WRITE_CALENDAR`。

官方参考：

- [Calendar Provider概览](https://developer.android.com/identity/providers/calendar-provider)
- [CalendarContract.Instances](https://developer.android.com/reference/android/provider/CalendarContract.Instances)
- [READ_CALENDAR权限](https://developer.android.com/reference/android/Manifest.permission#READ_CALENDAR)

### 8.1 Manifest

```xml
<uses-permission android:name="android.permission.READ_CALENDAR" />
```

运行时使用`ActivityResultContracts.RequestPermission()`申请。用户拒绝后仍允许生成日历，只在任务区域显示“未授权读取日历”，不要重复强制弹窗。

### 8.2 为什么查询Instances

应查询`CalendarContract.Instances`，不要直接只查`Events`：

- `Instances`会展开重复事件的每一次发生记录。
- 可以正确得到今天实际发生的实例。
- 查询URI必须附加开始和结束毫秒时间戳。

### 8.3 今天的时间窗口

必须根据手机当前时区计算本地自然日，不能简单使用`now ± 24小时`：

```kotlin
val zone = ZoneId.systemDefault()
val today = LocalDate.now(zone)
val beginMillis = today
    .atStartOfDay(zone)
    .toInstant()
    .toEpochMilli()
val endMillis = today
    .plusDays(1)
    .atStartOfDay(zone)
    .toInstant()
    .toEpochMilli()
```

### 8.4 查询示例

```kotlin
data class TodayEvent(
    val eventId: Long,
    val title: String,
    val beginMillis: Long,
    val endMillis: Long,
    val allDay: Boolean,
    val location: String?,
)

fun queryTodayEvents(
    resolver: ContentResolver,
    beginMillis: Long,
    endMillis: Long,
): List<TodayEvent> {
    val uriBuilder = CalendarContract.Instances.CONTENT_URI.buildUpon()
    ContentUris.appendId(uriBuilder, beginMillis)
    ContentUris.appendId(uriBuilder, endMillis)

    val projection = arrayOf(
        CalendarContract.Instances.EVENT_ID,
        CalendarContract.Instances.TITLE,
        CalendarContract.Instances.BEGIN,
        CalendarContract.Instances.END,
        CalendarContract.Instances.ALL_DAY,
        CalendarContract.Instances.EVENT_LOCATION,
        CalendarContract.Instances.STATUS,
    )

    val result = mutableListOf<TodayEvent>()
    resolver.query(
        uriBuilder.build(),
        projection,
        null,
        null,
        "${CalendarContract.Instances.BEGIN} ASC",
    )?.use { cursor ->
        val eventIdIndex = cursor.getColumnIndexOrThrow(
            CalendarContract.Instances.EVENT_ID)
        val titleIndex = cursor.getColumnIndexOrThrow(
            CalendarContract.Instances.TITLE)
        val beginIndex = cursor.getColumnIndexOrThrow(
            CalendarContract.Instances.BEGIN)
        val endIndex = cursor.getColumnIndexOrThrow(
            CalendarContract.Instances.END)
        val allDayIndex = cursor.getColumnIndexOrThrow(
            CalendarContract.Instances.ALL_DAY)
        val locationIndex = cursor.getColumnIndexOrThrow(
            CalendarContract.Instances.EVENT_LOCATION)
        val statusIndex = cursor.getColumnIndexOrThrow(
            CalendarContract.Instances.STATUS)

        while (cursor.moveToNext()) {
            if (cursor.getInt(statusIndex) ==
                CalendarContract.Events.STATUS_CANCELED) {
                continue
            }

            result += TodayEvent(
                eventId = cursor.getLong(eventIdIndex),
                title = cursor.getString(titleIndex)
                    ?.trim()
                    ?.takeIf { it.isNotEmpty() }
                    ?: "未命名日程",
                beginMillis = cursor.getLong(beginIndex),
                endMillis = cursor.getLong(endIndex),
                allDay = cursor.getInt(allDayIndex) != 0,
                location = cursor.getString(locationIndex)
                    ?.trim()
                    ?.takeIf { it.isNotEmpty() },
            )
        }
    }
    return result
}
```

实际调用前必须再次检查`READ_CALENDAR`权限，并在`Dispatchers.IO`执行查询。

### 8.5 任务排版规则

- 标题固定为“今日日程”。
- 全天事件显示`全天 标题`。
- 普通事件显示`HH:mm 标题`。
- 按开始时间升序排列。
- 最多显示5条。
- 标题最多两行，超出区域使用省略号。
- 超过5条时，最后一行显示`另有 N 项`。
- 不显示事件描述、参会人、会议链接和账号邮箱。
- 日志中不得输出事件标题和地点。

## 9. 跨日后的任务和天气过期

STM32能够自主更新日期和室内温度，但安卓绘制的任务和天气不会自动变化。

日历模板元数据至少保存：

```text
templateLocalDate
weatherValidUntilEpoch
templateCrc32
layoutVersion
```

推荐降级规则：

- RTC日期仍等于`templateLocalDate`：正常显示任务。
- RTC日期已经变化：STM32在重新流式输出模板时将任务区域覆盖为白色，并显示红色`SYNC`提示。
- 天气超过`weatherValidUntilEpoch`：将天气区域覆盖为白色并显示`SYNC`。
- 用户再次碰NFC后，安卓重新读取当天事件和天气并发送新模板。

如果希望跨日后仍显示正确任务，协议必须改为安卓发送未来多天的结构化事件，由STM32负责文字排版；这不属于本版本范围。

## 10. 定位和天气数据要求

Android系统没有统一的“系统默认天气服务”API。本项目指定
[Open-Meteo Forecast API](https://open-meteo.com/en/docs)作为默认天气提供商，
但仍通过接口隔离，方便以后替换服务商。

天气只在用户通过NFC进入日历生成流程、应用处于前台时获取一次位置，不持续跟踪，也不申请后台定位权限。城市级天气不要求GPS级精度；用户只授予近似位置时也必须正常工作。

### 10.1 Manifest权限

```xml
<uses-permission android:name="android.permission.INTERNET" />
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION" />
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />
```

不要声明：

```xml
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION" />
```

Android 12及以上允许用户只授予近似位置。若应用请求精确位置，必须把`ACCESS_FINE_LOCATION`和`ACCESS_COARSE_LOCATION`放在同一次运行时请求中；应用不能把“仅授予近似位置”视为错误。参考Android官方的[运行时定位权限说明](https://developer.android.com/develop/sensors-and-location/location/permissions/runtime)。

```kotlin
val locationPermissionLauncher = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { result ->
    when {
        result[Manifest.permission.ACCESS_FINE_LOCATION] == true -> {
            loadWeather(LocationAccuracy.PRECISE)
        }
        result[Manifest.permission.ACCESS_COARSE_LOCATION] == true -> {
            loadWeather(LocationAccuracy.APPROXIMATE)
        }
        else -> {
            loadCachedWeatherOrUnavailable()
        }
    }
}

locationPermissionLauncher.launch(
    arrayOf(
        Manifest.permission.ACCESS_FINE_LOCATION,
        Manifest.permission.ACCESS_COARSE_LOCATION,
    )
)
```

权限应在用户点击“生成日历”或NFC识别成功后、界面说明天气需要当前位置时申请，不能在应用无上下文启动时直接弹出。

### 10.2 获取一次当前位置

推荐使用Google Play services的`FusedLocationProviderClient`：

```kotlin
interface DeviceLocationRepository {
    suspend fun getCurrentLocation(): Result<DeviceLocation>
}

data class DeviceLocation(
    val latitude: Double,
    val longitude: Double,
    val accuracyMeters: Float?,
    val capturedAt: Instant,
)
```

获取策略：

1. 权限已经授予后调用`getCurrentLocation()`请求一次新位置。
2. 使用平衡功耗精度，天气不要求高精度GPS。
3. 新位置超时或返回`null`时尝试`lastLocation`。
4. 最近位置不得早于6小时；超过6小时视为不可用。
5. 定位完成后不注册持续位置更新。
6. 系统定位关闭时提示用户打开定位，或者使用天气缓存。

Android官方建议需要较新鲜的位置时优先使用`getCurrentLocation()`，而`lastLocation`速度快但可能过期：[获取当前位置](https://developer.android.com/develop/sensors-and-location/location/retrieve-current)。

### 10.3 Open-Meteo请求

默认请求：

```text
GET https://api.open-meteo.com/v1/forecast
  ?latitude={latitude}
  &longitude={longitude}
  &current=temperature_2m,weather_code
  &daily=temperature_2m_max,temperature_2m_min,precipitation_probability_max
  &timezone=auto
  &forecast_days=2
```

应用只需要当前天气和当天预报：

- `current.temperature_2m`：室外当前温度。
- `current.weather_code`：WMO天气代码，用于选择四色图标和中文描述。
- `daily.temperature_2m_min[0]`：当天最低温。
- `daily.temperature_2m_max[0]`：当天最高温。
- `daily.precipitation_probability_max[0]`：当天最大降水概率。
- `timezone=auto`：让天气结果使用坐标对应时区。

经纬度发送前建议四舍五入到小数点后两位，约为公里级精度，足以查询天气并减少不必要的位置精度暴露。

### 10.4 数据接口

```kotlin
data class WeatherSnapshot(
    val conditionText: String,
    val currentCentiC: Int,
    val lowCentiC: Int?,
    val highCentiC: Int?,
    val precipitationPercent: Int?,
    val icon: WeatherIcon,
    val observedAt: Instant,
    val validUntil: Instant,
)

interface WeatherRepository {
    suspend fun getWeather(
        location: DeviceLocation,
        forceRefresh: Boolean,
    ): Result<WeatherSnapshot>
}
```

要求：

- 默认使用Open-Meteo，不在UI中要求用户选择天气提供商。
- 必须同时支持精确位置和近似位置授权结果。
- 不申请后台定位，应用离开前台后不继续更新位置。
- API失败时优先使用未过期缓存。
- 没有可用缓存时显示“天气不可用”，仍允许发送日历。
- Open-Meteo服务条款、请求额度和商业使用条件需要在发布前复核；接口层必须允许以后切换自建代理或其他提供商。
- 明确区分“室外”天气温度和AHT20“室内”温度。
- 日志、崩溃报告和分析平台不得记录原始经纬度。

## 11. 安卓绘图实现

使用`Bitmap + Canvas + Paint`生成固定像素画面：

```kotlin
val bitmap = Bitmap.createBitmap(
    400,
    300,
    Bitmap.Config.ARGB_8888,
)
val canvas = Canvas(bitmap)
canvas.drawColor(Color.WHITE)
```

Android官方`Canvas`支持绘制文字、直线、矩形和Bitmap：
[Canvas API](https://developer.android.com/reference/android/graphics/Canvas)。

建议拆分：

```text
CalendarBaseRenderer
  drawGrid()
  drawWeekdayHeader()
  drawWeather()
  drawTodayEvents()
  drawSyncTimestamp()

CalendarPreviewOverlayRenderer
  drawDateNumbers()
  drawCurrentDateHeader()
  drawIndoorTemperature()
  drawTodayHighlight()
```

`CalendarBaseRenderer`的输出参与CRC/HMAC并发送；`CalendarPreviewOverlayRenderer`只作用于预览副本。

### 11.1 色板

| 色码 | 颜色 | RGB建议值 |
|---:|---|---|
| 0 | 黑 | `#000000` |
| 1 | 白 | `#FFFFFF` |
| 2 | 黄 | `#FFD700` |
| 3 | 红 | `#DC0000` |

日历UI模式关闭Floyd–Steinberg抖动，避免文字和细线产生噪点。天气照片类图标也应使用预先制作的四色图标。

### 11.2 字体

- MVP可以使用Android系统无衬线字体。
- 文字使用整数像素坐标。
- 网格、正文和图标在400×300原始尺寸上直接绘制，不先画大图再缩小。
- 若产品要求不同手机生成完全一致的字形，应打包经过授权的字体子集。
- 任务区域应按实际像素宽度测量和省略，不能按字符数量硬截断。

### 11.3 2bpp打包

基础模板最终量化为四色后按现有协议打包：

```text
byte = p0 << 6 | p1 << 4 | p2 << 2 | p3
```

- 每行100字节。
- 共300行。
- 总长度必须为30,000字节。
- `p0`是当前字节最左侧像素。

## 12. UI和状态

认证成功后的日历页面显示：

- 当前设备短ID。
- 室内温湿度读取状态。
- 系统日历权限状态。
- 定位权限及当前位置获取状态。
- 今天读取到的事件数量。
- 天气更新时间。
- 400×300最终模拟预览。
- “重新读取日程和天气”。
- “发送日历到墨水屏”。

发送按钮只有在以下条件满足时启用：

- BLE已认证。
- `TIME_SYNC`成功。
- 固件声明日历模板能力。
- 基础模板编码长度为30,000字节。
- 最终预览已经生成。

日历权限或天气失败不是致命错误，应用应显示对应占位内容后继续发送。

## 13. 建议代码模块

```text
calendar/
  CalendarPermissionController.kt
  SystemCalendarRepository.kt
  TodayEvent.kt
  CalendarLayoutV1.kt
  CalendarBaseRenderer.kt
  CalendarPreviewOverlayRenderer.kt

location/
  DeviceLocation.kt
  DeviceLocationRepository.kt
  FusedDeviceLocationRepository.kt

weather/
  WeatherRepository.kt
  OpenMeteoWeatherRepository.kt
  OpenMeteoWeatherApi.kt
  WeatherSnapshot.kt
  WeatherIconMapper.kt

device/
  IndoorReading.kt
  TimeSyncEncoder.kt
  CalendarTransferCoordinator.kt

image/
  FourColorQuantizer.kt
  FramePacker.kt

ui/calendar/
  CalendarScreen.kt
  CalendarViewModel.kt
  CalendarUiState.kt
```

耗时任务分配：

- 日历Provider查询：`Dispatchers.IO`。
- 单次前台定位：异步等待`FusedLocationProviderClient`结果。
- 天气网络请求：`Dispatchers.IO`。
- Canvas绘制、量化和打包：`Dispatchers.Default`。
- BLE GATT操作：沿用现有单写入队列，不能并发写。

## 14. 隐私和安全

- 只申请`READ_CALENDAR`，不要申请`WRITE_CALENDAR`。
- 定位仅用于天气查询；不申请后台定位权限，不持续跟踪。
- 向天气提供商发送前将坐标降低到约小数点后两位精度。
- 不持久化原始精确坐标，不在日志、崩溃报告或分析事件中记录坐标。
- 只查询当前自然日需要的事件实例。
- 不上传日历标题、地点或时间到天气服务。
- 不在日志、崩溃报告或分析事件中记录日历内容。
- 不缓存完整日历数据库；基础模板可以保留在内存直到传输完成。
- `TIME_SYNC`必须带会话HMAC，不能只依赖CRC32。
- 日历模板必须通过图片HMAC后才能被固件标记为有效。
- NFC pairing key和BLE session key继续遵循现有清理规则。

## 15. 验收测试

### 15.1 日历读取

- 无日历权限时能显示占位内容并继续发送。
- 普通事件、全天事件和重复事件都能显示。
- 已取消事件不显示。
- 跨越午夜和跨时区事件不会重复或遗漏。
- 事件超过5条时显示剩余数量。

### 15.2 绘图

- 输出严格为400×300和30,000字节。
- 网格线坐标与STM32日期坐标逐像素一致。
- 基础模板的日期、温度和高亮区域保持纯白。
- 最终预览不会把模拟叠加层写入发送数据。
- 文字和网格量化后没有灰色噪点。

### 15.3 定位和天气

- 精确位置授权时能够获取并显示天气。
- 仅授予近似位置时同样能够获取并显示天气。
- 拒绝定位权限时使用未过期缓存，否则显示“天气不可用”。
- 系统定位关闭、当前位置返回`null`和定位超时均不会导致应用崩溃。
- 请求中经纬度已降低精度，日志中不出现原始坐标。
- Open-Meteo超时、HTTP错误和字段缺失时能够降级。
- 应用进入后台后没有持续定位请求。

### 15.4 联调

- NFC、认证、传感器读取、校时和日历发送完整成功。
- 墨水屏显示的日期和手机本地日期一致。
- 室内温度来自AHT20，不是天气API。
- 今日高亮框落在正确星期列和周行。
- 手机断开后RTC跨日，STM32能自动移动高亮并更新日期。
- 跨日后旧任务区域按规则显示`SYNC`，不会继续冒充今日任务。
- 普通照片模式不写模板Flash，也不叠加任何日历字符。
- CRC或HMAC失败时模板不生效，设备不刷新未认证数据。

## 16. 当前待确认项

以下项目不阻塞安卓框架开发，文档采用了推荐默认值：

1. “今日任务”按系统日历事件实现；如果实际需要Google Tasks，需要新增独立需求。
2. 默认天气提供商确定为Open-Meteo，同时保留`WeatherRepository`抽象。
3. 天气使用一次前台定位；接受精确或近似位置，不申请后台定位。
4. 任务和天气跨日后由STM32遮盖并显示`SYNC`，等待下一次NFC同步。
5. 日历基础模板只保存一份，不保存历史图片。
6. 布局固定为本文的`CALENDAR_LAYOUT_V1`；任何坐标变化都必须递增布局版本并同步升级两端。

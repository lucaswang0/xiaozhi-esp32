# GPIO 连接说明

适用板型：`bread-compact-wifi-lcd`（ESP32-S3）。以下引脚均定义在 [config.h](config.h)。

## 音频（I2S，Simplex 模式）

| GPIO | 宏定义 | 功能 |
| --- | --- | --- |
| GPIO4 | `AUDIO_I2S_MIC_GPIO_WS` | 麦克风 I2S WS（声道选择） |
| GPIO5 | `AUDIO_I2S_MIC_GPIO_SCK` | 麦克风 I2S 时钟 |
| GPIO6 | `AUDIO_I2S_MIC_GPIO_DIN` | 麦克风 I2S 数据输入 |
| GPIO7 | `AUDIO_I2S_SPK_GPIO_DOUT` | 扬声器 I2S 数据输出 |
| GPIO15 | `AUDIO_I2S_SPK_GPIO_BCLK` | 扬声器 I2S 位时钟 |
| GPIO16 | `AUDIO_I2S_SPK_GPIO_LRCK` | 扬声器 I2S 帧时钟 |

> 若注释掉 `AUDIO_I2S_METHOD_SIMPLEX` 改用 Duplex 模式，则使用备用定义：
> WS=GPIO4、BCLK=GPIO5、DIN=GPIO6、DOUT=GPIO7。

## 显示屏（SPI，ST7789）

| GPIO | 宏定义 | 功能 |
| --- | --- | --- |
| GPIO47 | `DISPLAY_MOSI_PIN` | SPI MOSI 数据 |
| GPIO21 | `DISPLAY_CLK_PIN` | SPI 时钟 |
| GPIO40 | `DISPLAY_DC_PIN` | 数据/命令选择 |
| GPIO45 | `DISPLAY_RST_PIN` | 复位 |
| GPIO41 | `DISPLAY_CS_PIN` | 片选 |
| GPIO42 | `DISPLAY_BACKLIGHT_PIN` | 背光（PWM 调光） |

## 按键与 LED

| GPIO | 宏定义 | 功能 |
| --- | --- | --- |
| GPIO0 | `BOOT_BUTTON_GPIO` | BOOT 按键：启动中进入配网，运行中切换对话 |
| GPIO48 | `BUILTIN_LED_GPIO` | 板载单色 LED 状态指示 |
| GPIO39 | `VOLUME_UP_BUTTON_GPIO` | 音量加（已定义，当前板级代码未初始化） |
| GPIO37 | `VOLUME_DOWN_BUTTON_GPIO` | 音量减（已定义，当前板级代码未初始化） |
| — | `TOUCH_BUTTON_GPIO` | 未使用（`GPIO_NUM_NC`） |

## 电池电压检测（ADC）

| GPIO | 宏定义 | 功能 |
| --- | --- | --- |
| GPIO9 | `BATTERY_ADC_CHANNEL = ADC_CHANNEL_8`（`ADC_UNIT_1`） | 电池分压点电压采样 |

分压电路：

```
电池正极 ── 10kΩ（BATTERY_UPPER_RESISTOR）── GPIO9 ── 4.7kΩ（BATTERY_LOWER_RESISTOR）── GND
```

换算关系：

```
Vpin  = Vbat × 4.7 / (10 + 4.7)
Vbat  = Vpin × (10 + 4.7) / 4.7
```

- ADC 使用 12dB 衰减，采样后经校准换算为电压。
- 电压同时用于两处显示：状态栏电池图标（电量百分比）和图标旁的实时电压数值（如 `4.03V`）。
- 板上没有充电检测引脚，因此始终按放电状态处理。

## 其他

| GPIO | 宏定义 | 功能 |
| --- | --- | --- |
| GPIO18 | `LAMP_GPIO` | MCP 测试用灯控输出 |

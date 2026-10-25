# 🌡️ ESP32 小小气象站

[![CI](https://github.com/zhenglinheihei-bit/esp32-mini-weather-station/actions/workflows/ci.yml/badge.svg)](https://github.com/zhenglinheihei-bit/esp32-mini-weather-station/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)

基于 **ESP32 + DHT22** 的迷你气象站。设备在你家局域网里起一个 Web 服务，手机或电脑打开网页即可实时查看温湿度，还能顺手开关板载 LED。**不依赖任何云服务**，数据不出家门。

## ✨ 功能

- 🌡️ 实时温湿度采集（DHT22，2.5s 采样；读数失败自动保留上次有效值）
- 📱 自适应手机/电脑的暗色 Web 仪表盘，2 秒自动刷新
- 🔌 简洁 JSON API，方便接入 Home Assistant 等家庭自动化
- 💡 网页一键开关板载 LED
- 📡 支持 mDNS：直接访问 <http://esp32-weather.local>，不用记 IP
- 🔁 WiFi 断线自动重连
- ✅ GitHub Actions 在线编译，每次推送自动验证固件能否构建

## 🧰 硬件清单

| 器件 | 数量 | 说明 |
| --- | --- | --- |
| ESP32 开发板（ESP32-DevKitC / NodeMCU-32S 等） | 1 | 任意基于 ESP32-WROOM-32 的板子 |
| DHT22 温湿度传感器（AM2302） | 1 | 用 DHT11 需改一行代码，精度较低 |
| 10kΩ 电阻 | 1 | 数据脚上拉（三针模块版自带，可省） |
| 杜邦线、面包板 | 若干 | |

## 🔌 接线

| DHT22 引脚 | 接到 ESP32 |
| --- | --- |
| VCC | 3.3V |
| DATA | GPIO4 |
| GND | GND |

> 裸传感器需在 DATA 与 VCC 之间加 10kΩ 上拉电阻；三针模块版已内置，直接接线即可。
> LED 使用板载 LED（GPIO2），无需额外接线。

## 📁 目录结构

```text
esp32-mini-weather-station/
├── .github/workflows/ci.yml   # GitHub Actions 在线编译
├── include/secrets.example.h  # WiFi 配置模板（复制为 secrets.h 使用）
├── src/main.cpp               # 全部固件代码（服务 + 网页 + 传感器）
├── platformio.ini             # PlatformIO 构建配置
├── .gitignore
├── LICENSE
└── README.md
```

## 🚀 快速开始

### 方式一：PlatformIO（推荐）

1. 安装 [VS Code](https://code.visualstudio.com/) 和 [PlatformIO IDE 插件](https://marketplace.visualstudio.com/items?itemName=platformio.platformio-ide)
2. 克隆本仓库并用 VS Code 打开：
   ```bash
   git clone https://github.com/zhenglinheihei-bit/esp32-mini-weather-station.git
   ```
3. 复制 `include/secrets.example.h` 为 `include/secrets.h`，填入 WiFi 名称和密码
4. 数据线连接 ESP32，点击底部状态栏的 **→ (Upload)**，固件自动编译、烧录
5. 打开串口监视器（115200）查看设备 IP，或直接访问 <http://esp32-weather.local>

### 方式二：Arduino IDE

1. 安装 [Arduino IDE](https://www.arduino.cc/en/software)，在开发板管理器安装 **esp32 by Espressif Systems**
2. 在库管理器安装 **DHT sensor library**（会提示一并安装依赖 Adafruit Unified Sensor）
3. 新建 sketch，把 `src/main.cpp` 的全部内容复制进去（替换默认内容）
4. 把 `include/secrets.example.h` 里的两个 `#define` 粘贴到 `#include <DHT.h>` 之后，填好 WiFi
5. 选择正确的开发板与端口，点击上传

## ⚙️ 配置项

`src/main.cpp` 顶部集中了可调参数：

| 宏 | 默认值 | 说明 |
| --- | --- | --- |
| `DHTPIN` | `4` | DHT 数据引脚 |
| `DHTTYPE` | `DHT22` | 用 DHT11 时改为 `DHT11` |
| `HOSTNAME` | `esp32-weather` | mDNS 域名与设备主机名 |
| `LED_BUILTIN` | `2` | LED 引脚（板载） |

## 📖 API 文档

所有接口均为 GET，返回 JSON，已开启 CORS（`Access-Control-Allow-Origin: *`）。

### `GET /api/data`

```json
{
  "temperature": 26.4,
  "humidity": 58.2,
  "led": false,
  "rssi": -52,
  "heap": 233012,
  "uptime": "0d 01:23:45",
  "ip": "192.168.1.42"
}
```

> 传感器从未成功读到数据时 `temperature` / `humidity` 为 `null`。

### `GET /api/led?state=on|off|toggle`

控制板载 LED 并返回最新状态（`state` 省略时等同 `toggle`）。

```bash
# 打开 LED
curl "http://esp32-weather.local/api/led?state=on"
# 翻转 LED
curl "http://esp32-weather.local/api/led?state=toggle"
```

### 其他

- `GET /` — Web 仪表盘页面
- 未知路径返回 `404 Not Found`

## ❓ 常见问题

**Q：网页打不开 / 找不到 IP？**
看串口监视器（115200）里打印的 IP；Windows 下 mDNS（`.local` 域名）有时不可用，优先用 IP 访问。

**Q：温湿度一直是 `--`？**
检查接线（DATA 是否在 GPIO4）、上拉电阻，以及 `DHTTYPE` 与实际传感器是否匹配。DHT22 冷启动最多需要等几秒。

**Q：板载 LED 亮灯逻辑反了？**
部分开发板 LED 是低电平点亮，把 `main.cpp` 里 `handleLed` 和 `setup` 中的 `HIGH`/`LOW` 对调即可。

**Q：想换引脚 / 换传感器？**
见上文「配置项」。

## 📄 许可证

[MIT](LICENSE)

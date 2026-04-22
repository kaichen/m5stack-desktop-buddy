# M5StickS3 Hardware Roadmap

未利用或半利用的板载能力，按性价比排序。仅限板载，不含 Grove 外接。

## IMU（BMI270，已用 accel，gyro / temp / 中断空着）

### 1. 双击外壳 → P_HEART / 喂食
- **现状**：BMI270 自带 tap interrupt，目前仅用 accel 阈值做 shake。
- **方案**：软件路径 — 50ms 内 |Δaccel| 双脉冲，间隔 80–400ms 视作双击。或直接配置 BMI270 INT1 引脚走硬件中断（需绕过 M5Unified 直访寄存器）。
- **触发**：DISP_NORMAL 下双击 → 触发 P_HEART 一次性动画 + `statsOnFeed()`。
- **冲突**：与 shake 阈值需分层（shake 是持续大幅，tap 是单点尖脉冲）。

### 2. 倾斜滚动 transcript
- **现状**：B 键翻 `msgScroll`。单手不便。
- **方案**：DISP_NORMAL 且非 inPrompt 时，pitch > +20° 视作"向后看"→ scroll++；< -20° → scroll--。每 600ms 限速一格。
- **风险**：与 clockUpdateOrient 共用 accel，需排他（clocking 模式下不滚）。

### 3. 翻腕手势审批
- **现状**：approve/deny 必须按键，prompt 弹出后腾不出手很尴尬。
- **方案**：inPrompt 期间检测 250ms 内绕 Y 轴 >120° 旋转。顺时针 = approve，逆时针 = deny。
- **风险**：误触代价高（误批权限）。需"摇晃确认"二次手势，或仅作"approve"单向。
- **建议**：先做单向（翻一下 = approve once），deny 仍按 B。

### 4. 步数 / 活动度
- **现状**：fed 槽完全靠 token。
- **方案**：BMI270 内置 step counter，但 M5Unified 0.2.14 未暴露；改用 accel 峰值检测（窗口 1s，过零 + 峰值 > 1.3g 计一步）。
- **挂钩**：每 100 步 = 喂食一次，让 fed 跟物理活动也挂上钩。
- **存储**：步数日累计入 NVS（与 tama.tokensToday 同 reset 周期）。

### 5. IMU 温度
- **现状**：DEVICE 信息页缺温度行，注释还说"AXP192 die temp 不可用"。
- **方案**：`M5.Imu.getTemp()` 一行调用 → DEVICE 页加 "die temp" 行。
- **工时**：5 分钟。

## 麦克风（PDM，完全闲置）

### 6. 环境噪声等级
- **方案**：`M5.Mic.begin()` + 200ms 周期采样，算 RMS，归一化为 0–4 格。
- **位置**：DEVICE info 页加 "noise" 一行 + 五格条。或独立 ENV info 页。
- **代价**：占用 I2S 通道，~2KB DMA buffer。

### 7. 拍手唤屏
- **方案**：屏息状态下持续低速采样，检测窄峰冲击（< 30ms 上升、> 12dB 突变），双拍间隔 100–500ms。
- **价值**：脱手唤屏，符合"宠物对你叫"调性。
- **代价**：屏息态本来 100ms 一次 loop，麦克风轮询会增加电流；需评估待机功耗。

### 8. 大笑/呼喊 → P_HEART
- **方案**：连续 1.5s RMS 高于阈值 → 触发宠物 heart 动画 + 短叫一声。
- **挂钩**：可计入 fed（"它喜欢热闹"）。

## IR LED（GPIO 9 / RMT，唯一闲置发光器件，肉眼不可见）

### 9. 调试/隐蔽 attention 信号
- **方案**：P_ATTENTION 时朝天花板猛闪。手机相机能看到。
- **价值**：低，主要给开发者看时序。可以选做。

### 10. NEC/RC5 协议遥控
- **方案**：menu 加一项"toaster mode"，按键发 TV power off 码。
- **价值**：彩蛋级，与 buddy 调性弱关联。skip 默认。

## WiFi（ESP32-S3 自带，未启用）

### 11. NTP 自同步
- **现状**：RTC 必须由 desktop bridge 推送时间。脱离桌面后表盘不准。
- **方案**：连上 WiFi 后 `configTime()` + pool.ntp.org，每 6 小时同步一次。
- **依赖**：需要 SSID/password 输入路径。

### 12. OTA 升级
- **方案**：`ArduinoOTA.begin()`，长按 B 进入 OTA mode，屏幕显示 IP。
- **价值**：开发体验质变 — 免拔线刷固件。
- **依赖**：同样需要 WiFi 凭据。

### 13. 凭据下发协议
- **现状**：阻塞 11 / 12 的瓶颈。
- **方案 A（推荐）**：BLE 已经鉴权，加 `wifi_set` 命令通过 NUS 写入 NVS。Desktop 侧加一个"配置 WiFi"按钮。
- **方案 B**：开机长按 B 进入 SoftAP captive portal，浏览器配。需要塞一个 mini HTTP 服务，flash 占用 +100KB。
- **方案 C**：SmartConfig（手机端 ESPTouch app），用户体验最差。
- **建议**：A，最干净；先实现协议，11/12 自然落地。

## USB（CDC 已可反向读入）

### 14. 桌面侧 USB 优先通道
- **现状**：固件已通过 `_usbLine.feed(Serial, out)` 读取 USB CDC JSON，并与 BLE RX 共用解析路径。
- **方案**：桌面 bridge 在检测到同一设备的 USB CDC 端口时优先走 USB，BLE 作为配对和无线 fallback。
- **价值**：开发期和桌面固定摆放场景更低延迟、更稳；BLE 断开/未配对时仍可注入 state。

## 电源/UI

### 15. 充电进度动画
- **现状**：DEVICE 页有数字百分比，主屏无视觉提示。
- **方案**：USB 充电中，主屏顶部画 16×6 电池 glyph，按 `M5.Power.getBatteryLevel()` 填充，1 Hz 动效。
- **价值**：放桌面充电时一眼看出充满了没。

### 16. 满电完成提示
- **方案**：从 charging → full 转换瞬间，beep(1800, 60) + P_HEART 一次性。
- **代价**：1 行 if，免费搭车。

---

## 落地顺序建议

1. **先做无依赖的（一晚搞完）**：#5 IMU 温度、#14 USB 反向通道、#15/#16 充电 UI、#1 双击、#2 倾斜滚动。
2. **再做需要协议变更的**：#13 WiFi 凭据下发，再开 #11 NTP、#12 OTA。
3. **彩蛋/选做**：#7 拍手唤屏、#8 大笑反馈、#9 IR 闪烁、#10 IR 遥控、#3 翻腕审批（误触风险评估后定）。
4. **看实测意愿再做**：#4 步数活动度（需评估 accel 峰值检测对功耗影响）。

---

## 参考

- M5StickS3 引脚：IR_TX=GPIO9, MIC_DATA=GPIO34, MIC_CLK=GPIO0, IMU 走 I2C0
- BMI270 datasheet: 双击/单击/活动检测寄存器在 0x55–0x5F
- M5Unified API：`M5.Imu.getTemp()`, `M5.Mic.record()`, `M5.Power.getBatteryLevel()`

# 设备识别对照表

同堂如何把 Home Assistant 的实体识别成各种卡片形态。**识别不对时，优先去 HA 里修正实体的 `device_class` / 名称 / 所属设备**，重新「同步实体与房间」即可自动纠正——本表告诉你每种识别依据的是什么。

## 一、支持的实体域（domain）

| HA 域 | 卡片形态 | 主控 / 关键参数（来自实体 attributes） |
|---|---|---|
| `light` | 灯 | 竖向亮度滑杆；`brightness`、`color_temp_kelvin`（`min/max_color_temp_kelvin`）、`rgb_color` |
| `switch` / `input_boolean` | 开关 | HomeKit 式拨杆；`device_class: outlet` 显示为插座 |
| `input_number` | 数值辅助 | 数值滑杆；`min`、`max`、`step` |
| `input_select` | 选项辅助 | 段选或下拉；`options` |
| `input_text` | 文本辅助 | 文本输入；使用 HA `input_text.set_value` |
| `input_datetime` | 日期时间辅助 | 日期、时间或日期时间输入；使用 HA `input_datetime.set_datetime` |
| `climate` | 空调/温控 | 温度±与模式圆钮；`hvac_modes`、`fan_modes`、`min_temp`、`max_temp`、`current_temperature` |
| `fan` | 风扇/浴霸/新风 | 风速滑杆（有 `percentage` 时）或拨杆；`preset_modes`（模式段选）、`oscillating`（摇摆开关） |
| `cover` | 窗帘/车库门/卷帘 | 有 `current_position` → 开度滑杆；无 → 开/关拨杆。`device_class: garage/door/gate/window/shutter` 归入「安全」分区 |
| `lock` | 门锁 | 拨杆（上锁/解锁） |
| `alarm_control_panel` | 安防系统 | 竖排模式列（在家/离家/夜间/关闭）；`supported_features` 决定可用模式、`code_arm_required`/`code_format` 决定是否要密码 |
| `humidifier` | 加湿/除湿器 | 拨杆 + 目标湿度滑杆；`available_modes`、`min/max_humidity`、`humidity`（目标） |
| `valve` | 阀门/水龙头/灌溉 | 有 `current_position` → 开度滑杆；无 → 拨杆 |
| `vacuum` | 扫地机 | 开始/暂停/回充圆钮；卡片图标点击=清扫中回充、否则开始 |
| `lawn_mower` | 割草机 | 开始/暂停/回充圆钮 |
| `media_player` | 扬声器与电视 | 播放/暂停；归入「扬声器与电视」分区 |
| `camera` | 摄像头 | 首页快照卡（15 秒），点开 MJPEG 实时流 |
| `sensor` | 传感器 | 数值 + 24 小时走势线；分类见下文「传感器读数分级」 |
| `binary_sensor` | 二元传感器 | 见「人体存在识别」「门磁」；`device_class` 决定语义 |
| `button` / `input_button` | 一键操作 | 圆钮（浮窗「一键操作」区） |
| `number` | 数值调节 | 滑杆；`min`、`max`、`step` |
| `select` | 选项 | ≤6 个选项显示段选按钮，否则下拉；`options` |
| `remote` / `infrared` | 遥控器 | 发送 HA 遥控指令；`commands` 可生成快捷指令按钮 |
| `assist_satellite` | 语音卫星 | 显示卫星状态，可开始语音对话 |
| `air_quality` | 空气质量 | 显示当前指数/状态及可用读数 |
| `weather` | 天气 | 显示天气状态与温度 |
| `device_tracker` | 位置追踪 | 显示在家/离家及可用位置属性 |
| `siren` | 警报器 | 拨杆 |
| `water_heater` | 热水器 | 拨杆 |
| `event` | 事件 | 最近触发时间；`event_type` |
| `image` | 图像 | 图片面板（扫地机地图、打印封面等） |

不在上表中的域（如 `ai_task`、`conversation`、`calendar`、`notify`、`stt`、`tts`、`wake_word`、`todo`、`update`）不会同步为设备卡；情景/自动化、人员在家走各自的专门功能。

## 二、组合设备（自动合并成一张卡）

**依据 HA 设备注册表的 `device_id`**：同一物理设备下 ≥2 个实体自动折叠为一张组合卡。想让实体合并/拆分，在 HA 里调整实体的「所属设备」即可。

主控实体按优先级选取：`climate → fan → light → cover → humidifier → valve → lawn_mower → alarm_control_panel → switch → media_player → lock → vacuum → water_heater → siren`；全是传感器时优先名称含「实际/actual/current」的温度传感器，其次门磁/水浸/燃气/烟雾等安全类二元传感器。主控为开关且设备内有多个开关时，整卡变为「组合开关」（HomeKit 插排样式）。

**配置/诊断类实体自动降级**：HA 实体注册表标注了 `entity_category: config / diagnostic` 的实体（Zigbee2MQTT 的 Indicator 指示灯、Power outage memory 断电记忆、灵敏度校准、设备温度/电压等）**不参与**主控选取、开关计数、全开全关与总开关级联；控制项收进浮窗折叠的「设备配置」区，读数归入「更多信息」。若某配置项被当成了正常开关，在 Zigbee2MQTT/HA 中给该实体设置正确的 entity category 即可。

组合浮窗自动分区：**关键指标大数字**（前三，见下）→ 寿命与电量 → 一键操作 → 开关 → 调节 → 更多信息（默认折叠）。

## 三、特殊形态识别规则

### 人体存在传感器
- **判定**：`binary_sensor` 且 `device_class` 为 `occupancy` / `presence` / `motion`，**或名称含「人在 / 有人 / 存在」**
- 卡片显示「有人 / 无人 · 照度」，默认运动传感器图标；指示灯开关、灵敏度等配置项收进折叠的「设备配置」
- **多路网关**：一个设备下 ≥2 个存在实体、且各路名称前缀互不相同时，按前缀拆成多张卡（如「厨房人在」吸纳所有「厨房×××」实体）——请保持同一路实体统一前缀命名
- **分区型人在传感器**（小米人在传感器Pro 等支持区域划分的毫米波）：一个设备下 N 个分区占用实体前缀相同 → 合并为**一张卡**，任一分区有人即显示「有人」（多区时显示「有人 · N 区」），主控优先取名称含「整体/总体」的实体，各分区状态在浮窗「更多信息」查看
- 识别不对怎么办：在 HA 实体设置里把 `device_class` 改为 `occupancy`，或将实体重命名为「××人在」

### 3D 打印机
- **判定**：设备内同时存在名称含「进度 / progress」的数值传感器 + 名称含「暂停 / 停止 / pause / stop」的按钮
- 卡片显示「打印状态 · 进度% · 剩余时间」（自动寻找名称含「打印状态/当前阶段」「剩余时间/remaining」的传感器）

### 插排端口功率
- 组合开关卡内，功率/电流传感器（`device_class: power/current` 或单位 W/kW/A/mA）按两条规则挂到端口行：**传感器名称包含开关名称**，或**两者实体 ID 的纯数字尾号 / 名称中的数字一致**
- 没匹配上的（总功率、总电量）留在大数字区与更多信息

### 冰箱等纯传感器设备
- 主控取「实际温度」；任一 `device_class: door/opening` 的门磁为开时，卡片状态追加「N 门未关」

### 门磁 / 门窗
- `binary_sensor` 且 `device_class` 为 `door` / `window` / `opening` / `garage_door`：房间摘要条显示「接触式感应器：N 处开着 / 已关上」

### HomeKit 原生桥（家庭 App）补充识别
- **多键遥控拆分**：`event` 实体的 `event_types` 声明里出现 ≥2 个不同的数字键号前缀（Z2M 单实体多键，如 `1_single` / `2_double`）→ 家庭 App 拆成「按键 1..N」多个无状态按钮，可分别配自动化；识别不出键号结构（无前缀 / 单键号）维持单按钮
- **风扇自动档**：`fan` 实体 `preset_modes` 含 `auto` / `smart`（大小写不敏感）→ 家庭 App 显示「自动 / 手动」切换；切自动下发该实体的预设原词，切手动按当前风速回百分比（无风速实体则退回首个手动预设）
- **土壤湿度**：`sensor` 且 `device_class: soil_moisture`（Z2M 花草检测仪等）→ 湿度传感器（%）；仅原生桥候选，HA 自带桥不支持该类
- **安防布防码**：需要密码的 `alarm_control_panel`，在设备浮窗 → 设置 → 「布防码」录入（仅存本人账号）；家庭 App 布防/撤防会自动附带此码

## 四、传感器读数分级（浮窗展示位置）

| 位置 | 规则 |
|---|---|
| **大数字区（前 3 个）** | 按优先级：温度(`temperature`) → 湿度(`humidity`) → **照度(`illuminance` 或单位 lx)** → 功率(`power`) → ppm → PM2.5 → W/kW → 电量(`energy`)；名称含「设置/目标/setting/target」的自动让位给实测值 |
| **寿命与电量（进度条）** | 单位为 `%` 且（`device_class: battery` 或名称含「滤芯/寿命/滤网/filter/life/剩余」） |
| **更多信息（默认折叠）** | 其余所有只读实体 |

> 温湿度传感器带照度、人体存在传感器带照度，都会自动进大数字区。

## 五、房间视图的类别分区

摄像头 → 安全（锁/安防/警报器/车库门电动门窗/门磁烟感人体）→ 扬声器与电视 → 水（阀门/漏水）→ 温控（空调/风扇/热水器/加湿器/窗帘）→ 灯 → 电源与开关（开关/扫地机/割草机/按钮）→ 传感器与读数 → 其他。

## 六、常见纠正操作（在 HA 侧）

| 现象 | 修正方法 |
|---|---|
| 存在传感器被识别成开关/温度卡 | 实体 `device_class` 改为 `occupancy`，或重命名含「人在」 |
| 插座显示成普通开关 | switch 实体 `device_class` 改为 `outlet` |
| 组合卡缺实体 / 多实体没合并 | HA 实体设置中修改「所属设备」 |
| 端口功率没挂到对应端口 | 传感器命名包含端口开关名，或保持一致的数字尾号 |
| 传感器数值没进大数字区 | 补上正确的 `device_class`（temperature/humidity/illuminance/power…） |
| 卡片图标不合适 | 长按卡片 → 设置 → 更换图标（或在 HA 里设置实体图标，未自定义时自动继承） |

修改后回到管理后台点「**同步实体与房间**」，识别即时更新；HA 已删除的实体与区域会在同步时自动清理。

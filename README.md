# 同堂 Tongtang

> 「一家同堂」—— 多家庭、多用户、带精细权限的 Home Assistant 家庭控制台。
> HomeKit 风格界面，七套主题，支持跨多个 HA 实例统一管理。

**本仓库为发行仓库**：提供 Docker 镜像部署文件与完整文档，不包含源代码。
本项目不开源，保留所有权利，详见 [LICENSE](LICENSE)。问题反馈与需求建议请提 [Issues](../../issues)。

## 当前版本：2.4.9

本次更新降低 HA 状态流断线重连时的重复告警噪声：首次及每 10 分钟保留 WARNING，中间重试降为 DEBUG；故障仍会自动指数退避重连。

详细变更与升级注意事项见 [更新日志](CHANGELOG.md#249--2026-10-06)。

发布镜像：`jeesa/tongtang:2.4.9`、`jeesa/tongtang-web:2.4.9`、`jeesa/tongtang-api:2.4.9`、`jeesa/tongtang-mt:2.4.9`，支持 `linux/amd64` 和 `linux/arm64`，`latest` 已同步更新。HomeKit 原生侧车复用 API 镜像。

已有部署保留数据卷及 APP_SECRET，使用原 compose 配置执行 `docker compose pull`、`docker compose up -d`；一体化部署为两条命令均加上 `-f docker-compose.allinone.yml`。静态侧车应一并更新，更新前建议备份数据卷。

## 它解决什么问题

Home Assistant 很强，但它的权限模型是"全有或全无"——家里老人孩子一登录就能看到全部几百个实体，还能误删自动化。同堂在 HA 之上加了一层**面向家庭成员的控制台**：

- 管理员按人分配设备，每个人只看到、只能控制被授权的设备
- 界面完全对齐 Apple 家庭 App 的交互习惯，家人零学习成本
- 一套 Web 同时管理多个家庭、多个 HA 实例（自己家 + 父母家），默认一个家庭一个实例、互不混杂

## 快速开始

准备：一台能跑 Docker 的机器（NAS / x86 小主机 / 树莓派，amd64 与 arm64 均支持）+ Home Assistant 长期访问令牌。

**方式一 · 一体化单容器（最简）**

```bash
curl -O https://raw.githubusercontent.com/ZHonry/tongtang/main/docker-compose.allinone.yml
docker compose -f docker-compose.allinone.yml up -d
```

**方式二 · 双容器（Nginx 前端 + API 后端）**

```bash
curl -O https://raw.githubusercontent.com/ZHonry/tongtang/main/docker-compose.yml
docker compose up -d
```

**HomeKit 桥 · 原生侧车（设备整机进家庭 App，NAS/Linux 部署）**

两份 compose 文件里都预置了两种开通方式（注释开关，二选一）：

- **方式 A · 自动编排**：取消 `docker.sock` 挂载行的注释——成员第一次建桥时同堂自动创建 `tongtang-hk` 侧车容器（host 网络做 mDNS 广播、复用镜像与数据卷），桥全部删除自动回收，镜像升级后自动重建，全程零维护
- **方式 B · 静态侧车**：不愿挂 docker.sock 的用户，取消 `hk` 服务整段注释，由 compose 直接创建长驻侧车（api 端无需任何开关，未挂 docker.sock 时自动识别）

> **安全提示**：方式 A 挂载 docker.sock 等同于把宿主机管理权限授予容器，请自行评估信任边界；可经 [docker-socket-proxy](https://github.com/Tecnativa/docker-socket-proxy) 转发并只放行 containers 相关接口收紧，或直接选方式 B（完全不碰 docker.sock）。HomeKit/Matter 桥接需要 host 网络（NAS/Linux Docker）；Docker Desktop（Windows/macOS）不支持 host 网络，无法使用 HomeKit/Matter 桥接。

> **配对排障**：宿主机装有 WireGuard/Tailscale 等虚拟网卡或多网卡时，mDNS 可能通告错误 IP（表现为配对最后一步失败、已添加配件一直"未连接"）——设置 `HK_ADDRESS` 为 NAS 局域网 IP 即可（方式 A 设在 api 环境变量并自动透传，方式 B 设在 hk 服务，见 compose 注释）。桥端口被其他进程占用时会自动迁移到空闲端口，配对不受影响。iPhone 与服务器需同一网段（mDNS 不跨网段，跨段需路由器开 mDNS 反射）。

> **Zigbee2MQTT 用户注意**：① Z2M 的无线按钮/遥控器默认**不生成实体**——需在 Z2M 配置里开启 `homeassistant: experimental_event_entities: true` 后才能在同堂里勾选为 HomeKit 按钮；② Z2M 的智能插座官方不带 `device_class: outlet`，同堂会按名称（含"插座/plug"等）自动识别为插座形态，名字不含关键词的可在同堂"显示为"里手选，或在 Z2M 里 per-device 覆写 device_class。

**可选 · Matter 桥（一座桥同享苹果家庭 / Google Home / Alexa，NAS/Linux 部署）**

与原生 HomeKit 桥平行的第三种互联方式：每位成员自建 Matter 桥，凭 multi-admin 把同一座桥
同时配对给多个语音生态；iPhone 本机扫码即添加，无需 HomePod 中枢做桥接、无需 HA 的 Matter Server。
两份 compose 里同样预置两种开通方式（注释开关，二选一）：

- **方式 A · 自动编排**：api 环境变量取消 `MT_IMAGE` 注释并挂载 docker.sock（与 HomeKit 方式 A 共用同一行挂载）——成员建桥时自动创建 `tongtang-mt` 侧车（host 网络广播 `_matterc._udp`），桥删光自动回收，镜像标签更新后自动重建
- **方式 B · 静态侧车**：取消 `mt` 服务整段注释即可长驻，**api 端无需任何开关**（未设 MT_IMAGE 时同堂自动识别为常驻模式）

> **Matter 注意事项**：配对排障同 HomeKit（mDNS 不跨网段；虚拟网卡场景见上）。开源软件桥使用
> Matter 测试厂商码（0xFFF1），苹果家庭添加时会提示「未经认证的配件」，属预期行为，点继续即可。
> 同一宿主机部署多套同堂实例时，请为每套实例设置 `HK_SIDECAR_NAME` / `MT_SIDECAR_NAME`
> 错开侧车容器名（默认 `tongtang-hk`/`tongtang-mt` 为全局唯一，多实例共用会互相争抢）。
> 各生态对 Matter 空调的支持成熟度不同（社区实测）：苹果家庭/Google Home 支持温度与模式，
> **Alexa 目前对 Matter 空调仅支持开关**（不支持调温，系 Alexa 侧限制）；手机上的状态显示可能滞后
> （Google Home 尤甚，可达一两分钟）——指令实际即时生效，仅显示延迟，以设备实际动作为准。

打开 `http://服务器IP:8080`，按首次配置向导填入 HA 地址与长期访问令牌即可。
详细部署、升级、备份、HTTPS 与故障排查见 [docs/deployment.md](docs/deployment.md)。

镜像仓库：[`jeesa/tongtang`](https://hub.docker.com/r/jeesa/tongtang)（一体化） · [`jeesa/tongtang-web`](https://hub.docker.com/r/jeesa/tongtang-web) + [`jeesa/tongtang-api`](https://hub.docker.com/r/jeesa/tongtang-api)（双容器）

## 界面预览

> 以下截图来自真实家庭环境（约 700 实体规模）。

| 首页 · iOS 26 液态玻璃 | 首页 · 金门 Golden Gate |
|---|---|
| ![首页 iOS 26 玻璃](screenshots/home-glass.png) | ![首页 金门](screenshots/home-gold.png) |

| 庭院 · 新中式 | 夜航 · 仪表舱（暗色） |
|---|---|
| ![庭院新中式](screenshots/home-court.png) | ![夜航仪表舱](screenshots/home-nav-dark.png) |

| 果冻 · 多彩 | 素墨 · 黑白 |
|---|---|
| ![果冻多彩](screenshots/home-pop.png) | ![素墨黑白](screenshots/home-mono.png) |

| 能源账本（电/燃气/水 · 表计层级下钻） | 组合设备浮窗（插排端口功率） |
|---|---|
| ![能源页](screenshots/energy.png) | ![组合设备](screenshots/composite.png) |

| 情景与自动化 | 管理后台 · 实体管理 |
|---|---|
| ![自动化](screenshots/automations.png) | ![管理后台](screenshots/admin-devices.png) |

<p align="center"><img src="screenshots/mobile-home.png" width="320" alt="移动端首页"></p>

## 功能总览

### 控制体验（HomeKit 风格）
- **瓷贴卡片**：点图标即开关（各类型有语义：空调恢复上次模式、扫地机智能切换清扫/回充、影音播放暂停），点卡片/长按打开控制浮窗；方块/胶囊两种卡片样式
- **控制浮窗**：灯光竖向大滑条（亮度/色温/颜色）、空调大温度数字+模式圆钮、热水器水温与运行模式、安防面板布防/撤防、加湿器湿度滑条、窗帘/风扇/水阀开度滑条；风机支持预设模式（换气/干燥/取暖）与摇摆
- **设备自动组合**：同一物理设备的多个实体自动合并为一张卡（PDU 8 口 = 一张"3/8 开启"的组合开关卡），浮窗按类型分区：一键操作 / 开关 / 调节 / 更多信息
- **灯组**：一个设备多盏灯（双模灯/主灯+夜灯+氛围灯）合并为一张卡，浮窗组级调光调色（能力取并集、按各灯实际能力分发），每盏灯可点入单独精调
- **组合卡拆分/合并**（HomeKit「作为单独板块显示」）：组合卡一键拆成独立卡片、随时合并回去，按用户生效
- **「显示为」类型切换**（HomeKit Show As）：开关类实体可显示为 开关/插座/灯，图标、分类统计、房间分区联动
- **设备最近活动**：浮窗内 24 小时时间线（谁在几点做了什么 + HA 状态变化）
- **门铃响应**：按铃时顶部横幅 + 摄像头快照 + 一键看实时；页面后台时可选浏览器通知
- **低电量汇总**：任一设备电池 ≤20% 时首页出现「电量不足」入口
- **影音控制**：上一曲/播放暂停/下一曲/静音、音量滑条、信号源切换
- **配置项自动降级**：Zigbee2MQTT 等集成的指示灯、断电记忆、灵敏度校准等配置/诊断实体不抢卡片主控、不参与全开全关，收进折叠的「设备配置」区
- **智能形态**：插排端口行内嵌各口功率/电流；冰箱优先显示实际温度并提示「N 门未关」；3D 打印机卡片显示打印状态·进度·剩余时间；人体存在传感器统一显示「有人/无人 · 照度」
- **分区型人在传感器**：支持区域划分的毫米波（小米人在传感器Pro 等）自动合并为一张卡，任一分区有人即「有人」（多区显示「有人 · N 区」）；多路存在网关（涂鸦等）按前缀自动拆分为多卡
- **富详情**：关键指标大数字（温湿度/照度/TDS/功率）、滤芯与电池寿命进度条、传感器 24 小时历史走势线
- **摄像头**：首页快照区（15 秒刷新 + N 秒角标），点开即 MJPEG 实时流；扫地机地图、AI 抓拍图像直接展示
- **首页**：家名大标题、真实天气/日出日落、按拥有设备自动生成的分类胶囊（温控/灯/安全/扬声器与电视/水/用电/开启中）、谁在家状态、按房间分区
- **房间与分类双视图**（对齐 HomeKit）：房间页 = 状态摘要条（温湿度/空气/安防/各类开启数/门磁/动作/照度）+ 按类别分区；分类页 = 聚合指标 + 按房间分组
- **实时**：后端订阅所有 HA 实例的 state_changed 事件，WebSocket 按权限推送（300ms 批量合并），无轮询
- **个性化**：七套主题（iOS 17 经典 / iOS 26 液态玻璃 / 金门 Golden Gate / 庭院新中式 / 夜航仪表舱 / 果冻多彩 / 素墨黑白，玻璃类含浓度滑杆）、186 个内置图标 + 用户上传（单色图自动随主题着色）、用户头像
- **性能**：688 实体规模实测流畅——卡片记忆化按需重绘、折线图/摄像头进入视口才加载、后台标签页自动暂停轮询

### 互联平台（生态桥接）
- **小爱同学 · 巴法云**：各成员绑定自己的巴法云私钥，勾选授权内的设备同步进米家，小爱语音控制；权限收回自动撤出
- **涂鸦智能 · 虚拟网关**：管理员配一次 API Key，各成员扫码绑定专属网关，设备经涂鸦 App 确认后成为家庭顶层设备（完整产品面板、按房间自动归位），覆盖灯/开关/风扇/新风/窗帘/空调/热水器/加湿器/扫地机/温湿度/门磁，可接天猫精灵、小度等语音生态
- **HomeKit · 原生专属桥（侧车）**：每位成员自建多座桥（按房间/类型命名组织），iPhone 家庭 App 扫码配对、Siri 直控——**同一物理设备整机一块磁贴**（PDU 插排展开各口独立控制、温湿度电量合一、双模灯合并、新风→净化器卡片、热水器可接入），电视信源/遥控/音量、门铃与无线按钮原生通知、情景/脚本/自动化挂为一按即触发的开关；勾选与权限变化实时同步进家庭 App 且不破坏配对，权限收回自动消失，家庭 App 里的每次操作经同堂权限校验并记入审计（需 host 网络：NAS/Linux Docker 可用，Docker Desktop 不支持）
- **Matter · 专属桥（multi-admin，可选叠加）**：一座桥同时共享给**苹果家庭 / Google Home / Alexa**，iPhone 扫码直加（无需 HomePod 桥接、无需 HA Matter Server）；灯（开关/调光/色温/彩色）、插座、窗帘（含百叶倾角）、门锁、空调、热水器、风扇/净化器、水阀、全系传感器（温湿度/照度/气压/门磁/人体/水浸）、空气质量（PM2.5/PM10/CO₂/CO/NO₂/O₃/VOC/AQI 分级）、烟雾/燃气报警、情景触发全类型接入，控制经同堂权限校验并记入审计，权限收回实时消失
- 桥接均内置分步操作手册；小爱/涂鸦走 MQTT 长连接断线自愈、远端操作记入审计，HomeKit/Matter 为局域网直连（响应快、断网可用）

### 多家庭与权限
- **家庭 = 设备集 + 成员集**：授权、能源统计以家庭为边界强制校验
- **多 HA 实例**：附加实例即插即用（同步/实时/控制/摄像头/历史全自动覆盖）；添加实例自动创建同名家庭并绑定，同步后设备/房间自动归入——默认一个家庭一个实例；跨 HA 控制手动新建家庭挑选设备（带来源筛选）
- **用户多家庭**：一人可属多个家庭，侧栏一键切换，设备/家名/天气/能源/自动化随上下文切换
- **委托式管理**：普通用户可创建子成员，只能授予自己权限的子集，且自动互见在家状态
- **谁在家**：绑定 HA person 实体，显式互见关联才可见彼此状态（隐私默认关闭）

### 能源
- **电 / 燃气 / 水三本账**：日 / 周 / 月 / 自定义范围，构成环 + 按设备堆叠柱状图 + 明细与费用估算（可配单价）；数据走 HA 长期统计（与 HA 能源仪表盘同源）
- **表计层级**：总表 → 分表 → 挂接设备的树形分账，逐级下钻，父表差值显示「未计量」；表计配置给用户即成为其能源页根节点
- **三级作用域**：全局 / 家庭 / 个人（下级优先），家庭表自动授权全家

### 情景与自动化
- 复用 HA 原生引擎：新建情景（捕获当前状态一键还原）、自动化（时间/状态触发 + 多动作 + 延时 + 时间段条件），复杂逻辑可直改 HA 原生 JSON
- **按家庭隔离**：列表与创建跟随当前家庭所属的 HA 实例；普通用户只见自己创建的条目

### 管理后台
- 用户与权限（创建/编辑/重置密码/停用/家庭归属/人员绑定/互见关联）；授权面板支持实例/房间/类型筛选 + 筛选结果一键全选
- 实体管理（搜索 + 显示状态/类型/房间/响应状态/HA 实例筛选；无响应一键捞出；单个与多选批量隐藏/恢复）、设备分组（跨设备组合配件）
- 家庭管理（设备集/绑定实例/天气实体）、HA 实例管理（增删改/启停/连接测试）
- 能源配置（三级表计 + 层级树 + 单价）、审计日志（分页、保留时长自动清理）、HA 健康检查、数据备份接口

### 安全
- httpOnly Cookie 会话（SameSite=Lax + 自定义头防 CSRF）、登录限速、PBKDF2 密码
- HA 长期令牌仅存服务端；弱 APP_SECRET 检测；操作全量审计
- 镜像内为编译产物，不含可读源码；数据目录权限自愈（支持 PUID/PGID）

## 文档

| 文档 | 内容 |
|---|---|
| [docs/deployment.md](docs/deployment.md) | 部署、升级、备份恢复、HTTPS、安全清单、故障排查 |
| [docs/usage.md](docs/usage.md) | 使用手册：管理员工作流 + 家庭成员日常操作 |
| [docs/device-recognition.md](docs/device-recognition.md) | 设备识别规则：web 识别的设备形态对应 HA 的哪些参数，识别不对时如何在 HA 侧纠正 |
| [CHANGELOG.md](CHANGELOG.md) | 版本更新日志 |
| [THIRD-PARTY-NOTICES.md](THIRD-PARTY-NOTICES.md) | 第三方开源组件许可清单 |

## 支持的实体域

light · switch · climate · fan · cover · sensor · binary_sensor · camera · vacuum · media_player · lock · humidifier · valve · water_heater · alarm_control_panel · siren · lawn_mower · button · input_button · select · number · scene · automation · person · weather

## 已知限制

- 自动化引擎复用 HA 原生（Web 不自建执行引擎，跨 HA 联动在规划中）
- 能源历史依赖 HA 长期统计：实体需带 `state_class` 属性（详见部署文档故障排查）

## 支持项目

同堂是免费的业余项目，打赏纯属自愿，**不解锁任何功能**。如果它对你的家庭有用，可以请作者喝杯咖啡：

| 微信 | 支付宝 |
|:---:|:---:|
| <img src="docs/sponsor/wechat.png" alt="微信赞赏码" width="180"> | <img src="docs/sponsor/alipay.png" alt="支付宝收款码" width="180"> |

## 许可

Copyright © 2026. 保留所有权利（All Rights Reserved）。
本仓库内容仅供浏览与按文档部署官方发行镜像使用；未经书面许可，不得复制、修改或再分发。详见 [LICENSE](LICENSE)。

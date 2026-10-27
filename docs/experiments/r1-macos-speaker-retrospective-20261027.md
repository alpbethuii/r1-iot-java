# 斐讯 R1 作为 macOS 无线音箱：aptX LL、UDP 与 AirPlay 2 折腾复盘

记录日期：2026-10-27。状态：**自研低延迟电脑音箱路线退役；保留原厂普通蓝牙和已运行的 AirPlay 2。**

这份记录用于保存做过什么、哪些结论成立，以及哪些尝试不值得重复。它不是 aptX LL 固件发布，也不是已验收的低延迟音箱安装教程。未经完整验收的候选没有作为日用产品发布。

## 最终取舍

最初希望把闲置斐讯 R1 作为 MSI macOS / MiBook 的无线外接音箱，兼顾音乐、视频和游戏，且不能破坏“小禾”智能语音功能。先探索普通蓝牙和 aptX Low Latency，随后探索 CoreAudio 系统采集经 Wi-Fi UDP 直送 R1，最后打通 N1 AirPlay 2 接收加 Snapcast 的方案。

最终用户实际试听认为 R1 相比现有漫步者有线音箱，音质收益不足以支撑继续开发专用电脑音箱，因此决定停止 aptX LL 和自研 UDP 方向。AirPlay 2 的便利性和实际使用收益较高，继续保留。

**这是针对这套设备、约束和使用体验的工程止损，不是“已经证明 R1 硬件绝对不能实现低延迟或 aptX LL”。** 没有做标准化音质盲测，不能把主观差距全部归因于某颗芯片。

## 设备与保护范围

- 设备是**斐讯 R1**，不是华为 HiPlay 列表中其他品牌同名 R1。
- 实机：Rockchip RK322x，Android 5.1.1 / API 22，Build 3448。
- 活跃 Wi-Fi 固件和 NVRAM 对应 AP6255 / BCM43455C0。ROM 中有 Realtek 兼容库不代表实机装的是 Realtek；不能仅凭某个通用固件文件名确定蓝牙实际载入版本。
- 原厂 Android 已有 A2DP Sink，普通蓝牙接收无需再做一个“开启接收角色”的补丁。本项目未证明原厂具备 aptX LL。
- 保留原厂蓝牙、音频 HAL / DSP、音频策略、功放空闲行为，以及小禾的 APK、唤醒 / 收音和服务器后端。
- 实验无权因录音或 TCC 不通而绕过系统权限，也不能为降低延迟牺牲语音功能。

## 实验路线与真实成果

| 阶段 | 实际完成 | 没有完成 / 最终处理 |
| --- | --- | --- |
| 早期第三方接收方案 | 有过独立接收器 / Tethertone 等探索，后续已撤回 | 不作为最终运行依赖，不重新恢复 |
| 原厂蓝牙与缓冲研究 | 核实原生 A2DP Sink；同麦克风对照下 R1 相对有线音箱约 291–331 ms；512 / 128 帧 ABA 对照未证明降低 macOS I/O 缓冲有稳定收益 | 数字属于当时相对测量，不能推广为所有蓝牙设备的固定延迟 |
| aptX LL 软件与协议研究（10 月 18 日起） | libfreeaptx ARM / Android Bionic 构建，能力解析、11 / 20 字节配置桥、隔离 AOSP 修改；真机软件解码自测记录有 2000 个合成负载包 | 没有实际 LL 发射端、无线 GetCapabilities / SetConfiguration 联合证据，没有最终 LL 媒体播放和声学延迟成绩 |
| OEM HAL / 控制器研究 | 发现 Android 5.1 原始 AVDT_CODEC_SIZE=10 与 LL 基础配置长度冲突；追踪 OEM vtable、导出兼容、UART / 控制器释放与恢复 | 候选仍有约 100 个 OEM 导出缺口；恢复、实际 HCI 和 TX 资格不足，未把候选写入原厂系统 HAL |
| Java AudioTrack / OpenSL ES UDP | 验证独立接收、AudioFlinger 路径、队列、格式和采样率；积累受控短测 | 默认深缓冲和不同采样时钟等问题未因换 API 自动消失 |
| tinyalsa / 原生 UDP 接收 | 5 ms / 240 帧、48 kHz 双声道 S16；Focus 租约、预热握手、PCM 生命周期、漂移 / ASRC 与有界队列；局部启动排队由约 54.8 ms 降至 7.3 ms 的短时零 PCM 对照 | 局部排队改善不能从旧声学数字相减，宣称新端到端延迟 |
| macOS ScreenCaptureKit → CoreAudio Tap | Tap 实际非零系统音频；有界分包、独立网络线程、绝对节拍与调度；本机 30 秒 6000/6000 包证据，以及部分实机短测 | 合法 GUI / TCC 日用启动、长稳、正确的小禾插话后自动恢复和完整声学门没有闭环 |
| N1 AirPlay 2 → Snapcast → R1 | 真正 Mac AirPlay 会话；Chrome / Safari / QuickTime 实体录音录像各 11/11 事件；用户确认 iPad 同时向 MSI 与 R1 播放 | 日用保留；不是即时游戏方案。双端有轻微起声偏差，用户选择以后再校准 |

## aptX LL 为什么没有成为成品

aptX LL 需要发射端与接收端都真实支持，还需要 AVDTP 厂商能力协商、媒体分帧、解码、时钟 / 抖动管理和实际音频输出。**编译出解码库不等于蓝牙栈能协商 LL。** 普通 aptX、aptX HD、aptX Adaptive、FastStream 不能因名称相近而当作 aptX LL。

旧 AOSP 协议容量、原厂私有 ABI、控制器接管与恢复，以及尚未验证的发射端共同增加了工作量。部分离线门通过后，上线门仍应拒绝安装；没有用虚假导出或打开未完成 SEP 来掩盖这一点。

这里纠正一条曾出现过的错误论断：**“蓝牙芯片不是高通，所以软件实现 aptX LL 绝对不可能”不成立。** 主机软件解码的可行性与原厂是否提供 / 认证该功能是两件事。本次停止投入的理由是缺少完整链路、集成风险和实际收益，而非这个错误论断。

## UDP 方向留下了什么

这条方向验证了绕过蓝牙 codec 的网络 PCM 传输、Mac 真实系统采集、有界实时发送，以及 R1 短时 PCM 接收。工程里也处理过欠载、过期包、错误采样率、时钟漂移、预热 ACK、撤销租约和停止后的资源回收。

这些是可复用的组件知识，**不是已经做成的系统可选无线声卡**。日用候选一直未取得发布资格。特别是：

1. 传输收到零载荷包，不证明扬声器出声；日志中的写入成功也不替代实体听音。
2. 一次录像曾录到 MSI 原有有线音箱的提示声，而 R1 的握手 / PCM 写入为零。这次不能作为 R1 延迟成绩。
3. 绕过系统采集的直送短测约 147–149 ms，不等于“所有 Mac 系统声音到 R1 已低于 150 ms”。真实系统源曾出现约 186 ms 的单个合格声学窗口，仍有 XRUN / 稳定性问题，未通过整链验收。
4. 原厂 TTS 播放结束可能继续进入 ASR。TTS end、Focus 栈为空、静音都不能冒充“小禾整轮会话结束，可以恢复外部播放”的正向许可。
5. 空闲功放 Off 是可能的正常状态。测试脚本硬要求 On/On 会误拒绝；直接放宽守门或强制常开也不是正确的生命周期实现。

## 延迟测量中最值得保留的教训

- 系统自报 latency、环形缓冲容量、网络 RTT、收包到写入时间和物理声学延迟不是同一个量。
- 原厂 1024×6 / 48000 ≈ 128 ms 是缓冲容量；某次 delay 快照是瞬时积压，不能据此直接证明不可压缩的硬件下限。
- macOS 某一路由自报约 126 ms，不证明任何蓝牙音箱都必付这段时间。126+128+其他估计不能建立“物理天花板”。
- 周期性声画素材有相位歧义。每秒重复素材的 −420 ms 与 +580 ms 可能对应同一周期；必须用非周期事件和唯一频率匹配。
- 不同轮次、不同 API 和不同校准口径的测量不能直接相减宣布达标。
- Mac AudioContext 直接短音实验中，R1 相对同机有线输出新增延迟中位约 **2293 ms**（11 个配对事件）。这说明本套 AirPlay 链不适合即时交互；它不等于 iPad 同时投送两个接收器的同步偏差。
- 多设备同步应该用同一时间基准实测。用户听到 R1 略早于 MSI，但没有测得可靠毫秒数，因此不编造偏移参数。

## 保留下来的日常方案

**Apple 发送端 → N1 的 Shairport Sync（AirPlay 2）→ Snapserver PCM → Wi-Fi → R1 Snapdroid → Android 系统音频。**

本机使用过的版本是 Shairport Sync 5.6.2（AirPlay 2 编译）、NQPTP 1.2.9、Snapserver 0.31.0、Snapdroid 0.29.0.2。版本只描述此项目现场，并非通用升级建议。PCM 配置为 48 kHz / 16 bit / 2 声道，Snapcast 缓冲 200 ms、块 20 ms；开始和结束会话时控制接收器启停。

Snapcast 这一跳使用 PCM，不再加有损 codec；这不能扩张成“任何 AirPlay 音源到 DAC 全程 bit-perfect”，也不保证高解析音频原格式保持。AirPlay 的发送方式 / 编码和 Android 混音仍属于全链路的一部分。

N1 用有线连接网络，R1 保持 Wi-Fi。日常从 Mac / iPad 的 AirPlay 输出中选择接收器即可。iPad 可同时勾选 Mac 原生 AirPlay 接收端和 R1；这个组合已经用户实际确认出声。HiPlay、游戏低延迟、长时间插话共存不在已验收范围内。

## 退役和复用约定

- 自研 UDP、aptX LL、实验采集 App / 控制器 / 专用发送器不再作为 live 入口。
- 源码、实验报告、候选身份和必要私有证据归档；缓存、重复工作副本、编译暂存和过期票据按归属清理。
- 保留 AirPlay 2 运行时和独立智能语音工程，不通过“恢复原厂”覆盖小禾最新开发成果。
- 此仓库只保存脱敏经验；不上传现场录音录像、私有 APK / ROM、凭据、设备地址和原始日志。
- 将来复用先读本记录和归档索引，重新验证 OS / ABI / 权限 / 硬件条件，不能直接运行旧试验脚本。

## 上游参考

- [Shairport Sync](https://github.com/mikebrady/shairport-sync)
- [NQPTP](https://github.com/mikebrady/nqptp)
- [Snapcast](https://github.com/badaix/snapcast)
- [Snapdroid](https://github.com/badaix/snapdroid)
- [BlueZ codec 定义](https://github.com/bluez/bluez/blob/master/profiles/audio/a2dp-codecs.h)
- [libfreeaptx](https://github.com/regularhunter/libfreeaptx)
- [AOSP Android 5.1.1 Bluedroid](https://android.googlesource.com/platform/external/bluetooth/bluedroid/+/android-5.1.1_r9)
- [Apple Core Audio Tap](https://developer.apple.com/documentation/coreaudio/capturing-system-audio-with-core-audio-taps)

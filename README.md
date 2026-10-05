# win-notification

Windows 语音合成与声音提醒 Skill —— [mac-voice](https://github.com/Asterracy/mac-voice) 的 Windows 复刻版。

利用 Windows 内置能力实现语音播报、声音提醒、弹窗提醒和音量控制。**零第三方依赖**，全部使用系统自带命令与 .NET 内置类。所有能力经实测验证（见 SKILL.md「实测记录」章节）。

## 核心能力

| 能力 | Windows 实现 |
|------|-------------|
| 文字转语音播报 | `System.Speech.SpeechSynthesizer`（慧慧/康康/瑶瑶/Zira） |
| 系统提示音 | `System.Media.SystemSounds`（5 种） |
| 音量读取 | `winmm.dll waveOutGetVolume` P/Invoke |
| 通知横幅（非阻塞） | WinRT `ToastNotification`（Windows 10/11 右下角） |
| 警示弹窗（阻塞） | `System.Windows.Forms.MessageBox` |
| 输入对话框 | `Microsoft.VisualBasic.InputBox` |
| 超时弹窗（防卡死） | `WScript.Shell Popup`（原生支持 timeout） |
| 循环播报 + 监测 | PowerShell `while` 循环 + `Stop-Process` |

## 三分类提醒决策

- **分类 1 · 任务完成通知**：默认 Toast 横幅（非阻塞，自动消失）
- **分类 2 · 需人工操作（审批/扫码/登录）**：非静音→语音提醒；静音→后台警示弹窗；持续监测默认 60 秒，超时升级为循环语音
- **分类 3 · 个性化需求**：完全遵照用户 prompt 执行

## 发声授权规则（最高优先级）

只有用户在 prompt 中明确授权"可以叫我"，才能在任务执行中发声/弹窗呼叫。授权为**任务级**——仅绑定发出授权的那个任务，任务完成后自动失效。详见 SKILL.md。

## 与 mac-voice 的差异

- 音质：Windows 自带 SAPI 是传统合成音，不及 macOS 神经语音自然；需要时可升级 edge-tts（微软 Edge 神经语音，晓晓/云希，接近 Siri 级别）。`pip install edge-tts` 附带 `edge-tts`（合成文件）与 `edge-playback`（合成即播）两个命令，Windows 上 `edge-playback` 走 win32 原生播放，无需 ffmpeg/mpv。详见 SKILL.md「可选升级」一节
- 静音检测：Windows 无原生命令读静音状态，用音量 0% 近似判断
- Toast 横幅需 Windows PowerShell 5.1 执行（pwsh 7 加载 WinRT 类型失败）

## 安装

将 `SKILL.md` 放入你的 agent skill 目录，例如：

- `~/.opencode-core/skills/win-notification/SKILL.md`
- `~/.agents/skills/win-notification/SKILL.md`
- 或 `.claude/skills/win-notification/SKILL.md`（项目级）

## 相关

- [mac-voice](https://github.com/Asterracy/mac-voice) —— 本 skill 的 macOS 原版
- [SOUL.md personality guide](/concepts/soul)

## License

Apache-2.0

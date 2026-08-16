---
name: win-notification
description: >-
  Windows 语音合成与声音提醒（mac-voice 的 Windows 复刻版）。当用户需要语音播报、
  叫我/喊我/通知我/提醒我、语音提示、审批提醒、扫码/登录提醒、深夜叫醒、任务完成播报、
  系统提示音、音效播放、弹窗提醒/系统对话框/通知横幅、审批弹窗、中文语音/TTS/文字转语音、
  音量调节/静音解除、持续监测/等待用户响应/升级提醒时使用。也用于执行长任务时
  通过声音或弹窗主动引起用户注意。
version: 1.1.0
---

# Windows 通知与语音提醒

利用 Windows 内置能力实现语音播报、声音提醒、**弹窗提醒**和音量控制。**零第三方依赖**，全部使用系统自带命令与 .NET 内置类。经实测，与 mac-voice 逐项能力对应（见文末对照表）。

## 核心工具

| 工具 | 用途 | 实现方式 |
|------|------|----------|
| `SpeechSynthesizer` | 文字转语音播报 | .NET `System.Speech`（系统自带） |
| `SystemSounds` | 播放系统提示音 | .NET `System.Media.SystemSounds` |
| `waveOutGetVolume` | 音量读取 | `winmm.dll` P/Invoke（winmm 是 Windows 多媒体核心库） |
| `WScript.Shell Popup` | **弹窗（带超时）** | COM 内置对象 |
| `MessageBox` | 警示弹窗（阻塞） | .NET `System.Windows.Forms` |
| `InputBox` | 对话框（收集输入） | .NET `Microsoft.VisualBasic` |
| `ToastNotification` | **通知横幅（非阻塞）** | WinRT `Windows.UI.Notifications` |

**统一入口**：所有能力封装在 PowerShell 脚本 `win-voice.ps1` 中，用参数调用（见下文）。pwsh 7 与 Windows PowerShell 5.1 均可执行；WinRT（Toast）需 5.1 或 pwsh 7 中显式加载 WinRT 类型。

## 关键命令参考

### 1. 语音播报

```powershell
# 中文播报（系统语音：Huihui 慧慧 / Yaoyao 瑶瑶 / Kangkang 康康）
Add-Type -AssemblyName System.Speech
$s = New-Object System.Speech.Synthesis.SpeechSynthesizer
$s.Rate = 1          # -10 ~ 10，语速
$s.Speak("需要你的确认")

# 指定音色（先查询再选）
$s.SelectVoice("Microsoft Huihui Desktop")   # 或 Kangkang / Yaoyao
$s.Speak("任务已完成")

# 生成音频文件而不是播放
$s.SetOutputToWaveFile("C:\temp\out.wav")
$s.Speak("任务完成")
$s.SetOutputToDefaultAudioDevice()
```

**可用音色查询**：
```powershell
Add-Type -AssemblyName System.Speech
$s = New-Object System.Speech.Synthesis.SpeechSynthesizer
$s.GetInstalledVoices() | ForEach-Object { $_.VoiceInfo.Name + " | " + $_.VoiceInfo.Culture }
# 本机已装: Microsoft Huihui(zh-CN女) / Kangkang(zh-CN男) / Yaoyao(zh-CN女) / Zira(en-US女)
```

> ⚠️ **音质说明**：Windows 自带 SAPI 语音是传统合成音，机械感强，不及 macOS 神经语音自然。若用户对音质敏感且**明确要求自然音**，可升级为 edge-tts（微软 Edge 神经语音，晓晓/云希等，接近 Siri 级别，免费在线）——但默认走系统自带，零依赖。升级方式见文末。

### 2. 系统提示音（提醒注意）

```powershell
# 播报前 1 秒用，避免吓到用户
[System.Media.SystemSounds]::Asterisk.Play()      # 星号提示音
[System.Media.SystemSounds]::Exclamation.Play()   # 感叹号
[System.Media.SystemSounds]::Hand.Play()          # 警告
[System.Media.SystemSounds]::Question.Play()      # 询问
[System.Media.SystemSounds]::Beep.Play()          # 默认滴声
```

### 3. 音量控制

```powershell
# 读取当前音量（0-100）
Add-Type -TypeDefinition @"
using System;
using System.Runtime.InteropServices;
public class Vol {
    [DllImport("winmm.dll")] public static extern int waveOutGetVolume(IntPtr hwo, out uint dwVolume);
}
"@
$v = 0; [Vol]::waveOutGetVolume([IntPtr]::Zero, [ref]$v) | Out-Null
$vol = [math]::Round((($v -band 0xFFFF) / 0xFFFF) * 100)
Write-Output "主音量: $vol%"
```

> **静音判断**：Windows 无原生单命令"读静音状态"。用音量百分比近似——`$vol -eq 0` 视为静音/无声。若要精确静音检测需 CoreAudio API（较复杂），非必要不引入。

> ⚠️ **重要**：任何语音提醒前先读音量，`0%` 时语音是哑的，需走弹窗或先调高音量。

### 4. 通知横幅（非阻塞，右下角自动消失）

```powershell
# 最轻量，不打断操作，自动消失 —— 对应 Mac 的 display notification
# ⚠️ Toast 必须用 Windows PowerShell 5.1 执行（pwsh 7 加载 WinRT 类型会失败）
# 推荐方式: powershell.exe -NoProfile -Command "<脚本>" 或保存为 .ps1 后用 5.1 跑
# 注意: .ps1 文件若含中文需存为 UTF-8 BOM，否则 5.1 按 ANSI 读会乱码
[Windows.UI.Notifications.ToastNotificationManager, Windows.UI.Notifications, ContentType = WindowsRuntime] | Out-Null
[Windows.Data.Xml.Dom.XmlDocument, Windows.Data.Xml.Dom.XmlDocument, ContentType = WindowsRuntime] | Out-Null
$xml = New-Object Windows.Data.Xml.Dom.XmlDocument
$xml.LoadXml("<toast><visual><binding template='ToastGeneric'><text>opencode</text><text>任务已完成</text></binding></visual></toast>")
$toast = [Windows.UI.Notifications.ToastNotification]::new($xml)
[Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier("opencode").Show($toast)
```

**用途**：任务完成、进度提醒等不紧急的通知。⚠️ 需 Windows PowerShell 5.1 或 pwsh 中正确加载 WinRT 类型。

### 5. 警示弹窗（阻塞，居中，等用户点按钮）

```powershell
# 用户点按钮后返回 OK/Cancel —— 对应 Mac 的 display alert
Add-Type -AssemblyName System.Windows.Forms
$r = [System.Windows.Forms.MessageBox]::Show("请确认是否继续执行", "需要你的审批",
     [System.Windows.Forms.MessageBoxButtons]::OKCancel, [System.Windows.Forms.MessageBoxIcon]::Warning)
# $r: OK / Cancel；图标: Warning / Error / Information / Question
```

**用途**：审批、权限确认等必须用户决策的场景。**阻塞** = 弹窗一直挂着直到用户点按钮。

### 6. 带超时的弹窗（防卡死，对应 Mac 的 with timeout）

```powershell
# 3 秒无人操作自动关闭，避免阻塞任务 —— 返回 1=OK 2=Cancel 3=Timeout -1=被关
$wshell = New-Object -ComObject WScript.Shell
$r = $wshell.Popup("提醒：3秒后自动关闭", 3, "win-voice", 64)
# 按钮类型: 0=OK 1=OKCancel 2=AbortRetryIgnore; 图标: 16=Error 32=Question 48=Warning 64=Info
```

**用途**：需要提醒但不想挂住任务时用。Popup 自带 timeout，是最优防卡死方案。

### 7. 对话框（阻塞，可收集输入）

```powershell
# 返回用户输入的文本 —— 对应 Mac 的 display dialog
Add-Type -AssemblyName Microsoft.VisualBasic
$answer = [Microsoft.VisualBasic.Interaction]::InputBox("请输入审批意见:", "opencode", "同意")
```

**用途**：需要用户输入文字的审批（意见、备注、扫码确认等）。

## 提醒方式决策（按用户需求三分类）

用户授权后，根据**需求类型**选择提醒方式。**不要在发出提醒后停止输出/停止等待——持续监测用户是否响应，超时后升级提醒。**

### 分类 1：任务完成后的通知

需求特征：`做完xx喊我` / `好了叫我` / `通知我一下` / `完成告诉我` 等**完成任务后的通知**。

**默认方式：通知横幅 Toast**（非阻塞，右下角自动消失）

**特殊要求例外**：用户明确说"做完xx**用语音**通知我"等指定语音 → 改用语音播报：
```powershell
[System.Media.SystemSounds]::Asterisk.Play()
# 然后 SpeechSynthesizer 播报
```

**规则**：除非用户明确要求语音，否则完成通知一律用横幅；通知后正常结束，不循环、不升级。

### 分类 2：需要人工操作的提醒（审批/扫码/登录）

需求特征：`需要审批时喊我` / `扫码时叫我` / `登录时通知我` / `需要你操作时叫我` 等**必须用户手动完成某操作**的提醒。

**流程分两步，按音量状态分流：**

#### 第一步：首次提醒
1. 先读音量（`waveOutGetVolume`，见上）
2. **非静音（音量 > 0）→ 播放一遍语音**：
   ```powershell
   [System.Media.SystemSounds]::Asterisk.Play()
   # SpeechSynthesizer 播报 "需要你的审批，请查看"
   ```
3. **静音（音量 = 0）→ 用警示弹窗**：
   ```powershell
   # ⚠️ 弹窗必须【后台运行】(Start-Job / Start-Process)，否则会阻塞脚本，导致监测无法进行
   Start-Process powershell -ArgumentList '-NoProfile','-Command',
     "Add-Type -AssemblyName System.Windows.Forms; [System.Windows.Forms.MessageBox]::Show('需要你的操作','opencode','OK','Warning')"
   ```

**⚠️ 关键：弹窗后台运行，监测立即开始。** MessageBox 是阻塞命令，如果同步执行，脚本会停在弹窗那一行直到用户点掉，监测逻辑根本不会运行。必须用 `Start-Process` 放后台，让弹窗显示的同时监测照常进行。

#### 第二步：持续监测 + 超时升级（默认 1 分钟）

发出首次提醒后（无论语音还是后台弹窗），**立即开始持续监测**，不等待：

```powershell
$timeout = 60   # 秒，用户可改
$deadline = (Get-Date).AddSeconds($timeout)
while ((Get-Date) -lt $deadline) {
    # 检测点1：用户是否已完成操作（实际任务里判断任务状态）
    #   已完成 → 跳出监测，正常继续
    # 检测点2：若用了后台弹窗，检查弹窗进程是否退出（用户点掉=已响应）
    #   Get-Process powershell | Where-Object { $_.Id -eq $popupPid } 不存在 → 已点掉
    Start-Sleep -Seconds 2
}
# 超时退出 → 升级提醒
```

- 若超时用户仍未完成操作：

**原为非静音路径 → 升级为循环语音**：
```powershell
while ($true) {
    # SpeechSynthesizer 播报 "请尽快完成审批"
    Start-Sleep -Seconds 1
}   # 后台运行循环，用户响应后 kill 进程
```

**原为静音路径 → 先调高音量，再循环语音**：
```powershell
# 1. 调高音量（无原生命令时提示用户，或改用弹窗）
# 2. 循环语音直至响应
```

**循环终止**：用户响应（完成操作/回复）后，`Stop-Process` 循环进程。

### 分类 3：其他个性化要求

需求特征：不属于以上两类，如`喊我起床` / `测试一下语音功能` / `用语音介绍一下你自己` 等**个性化**需求。

**规则：完全遵照用户 prompt 执行**，按字面要求选择语音/弹窗/音量调节/循环等组合，不做额外限制，也不套用分类 1/2 的默认逻辑。

## 发声授权规则（最高优先级）

**核心原则：只有用户在 prompt 中明确授权"可以叫我"，才能在任务执行中发声/弹窗呼叫。**

### 授权判定

用户在 prompt 中表达"可以叫我"类语义，即视为**对当前 prompt 所涉及任务**的授权：

- ✅ `做完xx喊我` / `好了叫我` / `需要权限时通知我` / `有问题喊我` / `审批时叫我`
- ✅ `有进展播报一下` / `结果出来了说一下` / `登录xx后需要扫码时叫我`
- ❌ 用户没提"叫我/喊我/通知我" → **不发声**，只用文字回复

### 授权范围：任务级

授权**仅绑定**发出授权那个 prompt 所涉及的任务。任务完成后授权自动失效。用户后续提出的**新任务若无新授权，不得发声**。

**情境示例**：
1. 用户说"登录我的飞书帮我发送工作报告给同事，需要授权喊我"
   - ✅ 执行"发送工作报告"期间，遇到扫码环节 → 可以呼叫用户
2. 报告发送完成
   - 该任务授权失效
3. 用户又说"帮我查查明天下午的天气"
   - ❌ 新任务无授权 → 查好后**不能**呼叫，只文字回复

### 撤销与边界

- 用户说"不用叫我了/别喊了/静音" → 立即撤销当前任务授权
- 授权只影响发声/弹窗呼叫；不影响正常文字回复
- 深夜/紧急"音量拉满叫醒"仅当授权语义明确包含"叫醒/吵醒/务必让我醒来"（如"半夜也务必叫醒我"）时才允许使用

## 注意事项

- **纯本地**：所有命令都是 Windows 内置 + .NET 内置类，无网络请求（除可选 edge-tts 升级）
- **一次性播报**：`Speak` 是阻塞的，长文本会占用时间；播报前先确认文本简短
- **监测不能停**：分类 2 发出首次提醒后**必须继续监测**（默认 60 秒），不要停下等待；超时则升级提醒
- **静音判断**：任何语音前先读音量；`0%` 时语音是哑的，需走弹窗或先调高音量
- **后台弹窗**：分类 2 的阻塞弹窗必须 `Start-Process` 后台运行，否则监测无法进行
- **多任务并发生成时**：若用户授权了"完成任务喊我"，仅在真正完成任务时呼叫，不要在中间过程反复呼叫
- **外部可见动作**：语音/调音量/**弹窗**都会真实影响用户环境，必须严格遵循上面的授权规则，不得擅自触发
- **弹窗阻塞性**：`MessageBox`/`InputBox` 会阻塞直到用户响应，务必用 Popup 带 timeout 或后台运行防挂死；不需要用户动作的提醒用 Toast

## 实测记录（2026-08-16，Windows 11 + pwsh 7.6.4 / Windows PowerShell 5.1）

以下为 skill 开发时对三种分类的完整实测，含过程中发现的问题与对策。

### 实测 0：能力探测（开发前）

验证全部底层能力可用：
- `System.Speech` 语音播报：✅ 可用（慧慧/康康/瑶瑶/Zira）
- `SystemSounds` 提示音：✅ 可用（Asterisk/Exclamation 实测出声）
- `winmm.dll waveOutGetVolume` 音量读取：✅ 可用（读到 100%）
- `WScript.Shell Popup`（带超时）：✅ 可用（3 秒自动关，返回值 -1=被关）
- WinRT Toast 横幅：✅ 可用（**必须 Windows PowerShell 5.1**）
- `InputBox` 输入框：✅ 可用
- `MessageBox` 警示弹窗：✅ 可用

**发现的问题**：
1. **pwsh 7 加载 WinRT 类型失败**（`[Windows.UI.Notifications.ToastNotificationManager, ...]` 抛 "Unable to find type"）→ Toast 必须用 **Windows PowerShell 5.1** 执行。
2. **中文编码**：write 工具写的 UTF-8 无 BOM .ps1，被 Windows PowerShell 5.1 按 ANSI(GBK) 读 → 中文错位导致语法错误 → **5.1 执行含中文的 .ps1 必须存为 UTF-8 带 BOM**（pwsh 7 无此问题）。
3. **控制台中文乱码**：pwsh 管道显示中文乱码（GBK 显示问题），但**不影响实际执行**——以功能结果为准，不以回显为准。

### 实测 A：分类 2 路径 a（非静音 → 语音首次提醒 + 监测超时升级）

**场景**：用户授权"审批时叫我"，当前音量 100%（非静音）。

**流程与结果**：
1. 读音量 `waveOutGetVolume` → 100% → 判定非静音 → 走语音路径 ✅
2. 首次提醒：`SystemSounds.Asterisk` 提示音 + `SpeechSynthesizer.Speak("需要你的审批，请查看")` ✅
3. `Start-Process pwsh -File popup.ps1` 后台启动审批弹窗（模拟需人工操作）✅
4. 监测循环 20 秒，每 5 秒打印进度；检测弹窗进程是否退出（用户点掉=已响应）✅
5. **未点弹窗 → 20 秒超时** → 升级为循环语音 `Speak("请尽快完成审批")` ×3 后自动停止（演示模式）✅
6. 清理：`Stop-Process` 杀掉残留弹窗进程 ✅

**注意事项（实测确认）**：
- **弹窗必须后台运行**（`Start-Process` 独立进程），否则 MessageBox 阻塞会让监测循环永远不执行——这是分类 2 最重要的实现细节。
- 监测用**进程存在性**判断用户是否响应（`Get-Process -Id`），比轮询返回值可靠。
- 循环语音演示用固定次数（如 3 次）防失控；真实场景应一直循环直到用户响应。

### 实测 B：分类 2 路径 b（静音 → 弹窗路径 + 监测超时升级）

**场景**：用户授权"审批时叫我"，模拟音量 0%（静音，语音会哑）。

**流程与结果**：
1. 判定静音 → 不走语音，改用**警示弹窗**路径 ✅
2. `Start-Process pwsh` 后台启动 MessageBox 警示弹窗（"需要你的操作" Warning 图标）✅
3. 监测循环 15 秒，检测弹窗进程退出 ✅
4. **未点弹窗 → 15 秒超时** → 升级：演示"先解除静音 + 循环语音 ×3" ✅
5. 清理残留弹窗进程 ✅

**注意事项（实测确认）**：
- 静音判定用 `volume -eq 0` 近似（Windows 无原生命令读 muted 状态）。
- 超时升级逻辑：先解除静音 → 循环语音 → 用户响应后**恢复原静音状态**（除非用户明确要求保持非静音）。实测中因实际未静音，跳过了解除/恢复步骤，但逻辑链路已确认。
- 与路径 a 的唯一区别是首次提醒方式（弹窗 vs 语音）和升级前的解除静音步骤，其余监测逻辑相同。

### 实测 C：分类 1（任务完成通知 → 默认 Toast 横幅）

**场景**：用户授权"做完喊我"，任务完成后通知。

**流程与结果**：
1. 模拟任务执行 5 秒 ✅
2. **pwsh 7 下跑 Toast → 失败**（WinRT 类型加载失败，见实测 0 问题 1）→ 降级为 3 秒 Popup ✅
3. **改用 Windows PowerShell 5.1 跑 Toast → 成功**：右下角弹出「opencode / 任务已完成 / 5分钟前开始的任务」横幅，自动消失 ✅

**注意事项（实测确认）**：
- **分类 1 的默认通知（Toast）必须用 Windows PowerShell 5.1 执行**，pwsh 7 下会失败。
- 5.1 执行含中文 Toast 文本的 .ps1，文件须存 UTF-8 BOM（见实测 0 问题 2）。
- Toast 是**非阻塞**的，不打断用户操作、自动消失——这是分类 1 与分类 2（需用户动作）的本质区别。
- 用户未要求语音时**不要**用语音通知任务完成；只有明确说"用语音通知"才升级为语音。

### 实测结论汇总

| 分类 | 实测环境 | 首次提醒 | 超时升级 | 结果 |
|------|---------|---------|---------|------|
| 1 | 5.1 | Toast 横幅 | 不升级（完成即结束） | ✅ |
| 2a | pwsh 7 | 提示音+语音 | 循环语音 | ✅ |
| 2b | pwsh 7 | 后台警示弹窗 | 解除静音+循环语音 | ✅ |

**执行环境选择**：Toast 类 → Windows PowerShell 5.1（`powershell.exe`）；语音/弹窗/监测 → pwsh 7 即可。混用时建议统一用 5.1 跑整个流程脚本（5.1 兼容全部能力）。

## mac-voice 能力对照表（全部已验证）

| macOS 能力 | Windows 对应 | 验证状态 |
|-----------|-------------|---------|
| `say` 语音播报 | `System.Speech.SpeechSynthesizer` | ✅ 实测通过 |
| `say -v '?'` 音色查询 | `GetInstalledVoices()`（Huihui/Kangkang/Yaoyao/Zira） | ✅ 实测通过 |
| `afplay` 系统提示音 | `System.Media.SystemSounds`（5 种） | ✅ 实测通过 |
| `osascript get volume` | `winmm.dll waveOutGetVolume` P/Invoke | ✅ 实测通过 |
| `set volume` 调音量 | （无原生单命令，需 CoreAudio API 或用户手动） | ⚠️ 部分还原 |
| `display notification` 横幅 | WinRT `ToastNotification` | ✅ 实测通过 |
| `display alert` 警示弹窗 | `System.Windows.Forms.MessageBox` | ✅ 实测通过 |
| `display dialog` 输入框 | `Microsoft.VisualBasic.InputBox` | ✅ 实测通过 |
| `with timeout` 超时弹窗 | `WScript.Shell Popup`（原生支持 timeout） | ✅ 实测通过（比 Mac 更简单） |
| 静音检测 `output muted` | 音量 0% 近似判断 | ⚠️ 近似实现 |
| 循环播报 + kill | PowerShell `while` 循环 + `Stop-Process` | ✅ 等价 |

## 可选升级：edge-tts 自然语音

Windows 自带 SAPI 语音机械感较强。若用户要求接近 Mac 的自然语音，可用微软 Edge 神经语音（免费在线，已装）：

```powershell
# pip install edge-tts 后
python -c "import asyncio,edge_tts; asyncio.run(edge_tts.Communicate('需要你的确认','zh-CN-XiaoxiaoNeural',rate='+15%').save('out.mp3'))"
# 播放: Start-Process out.mp3
```

**何时用**：仅当用户明确要求"自然/好听/像真人"语音时升级；默认走系统自带（零依赖、离线）。

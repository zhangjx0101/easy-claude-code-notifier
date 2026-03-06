# Easy Claude Code Notifier

为 Claude Code（VSCode 扩展 / CLI）添加系统弹窗通知。当 Claude 完成任务、需要授权或提问时，弹出原生系统通知。

**无需安装任何第三方插件**，仅依赖 Claude Code 内置的 Hooks 机制 + 平台原生脚本。

## 平台支持

| 平台 | 状态 |
|---|---|
| Windows 10/11 | 已实现（Toast 通知） |
| macOS | 计划中 |

---

## 功能概览

| 事件 | 通知内容 | 状态 |
|---|---|---|
| Claude 完成回复 (Stop) | "Claude has finished the task." | 稳定工作 |
| Claude 需要工具授权 (PermissionRequest) | "Claude needs permission to use Bash." | 稳定工作 |
| Claude 向用户提问 (PreToolUse) | "Claude is asking you a question." | 不稳定（见 [已知问题](#已知问题)） |

---

## 前置条件

- Windows 10 (Build 10240+) 或 Windows 11
- Claude Code 已安装（VSCode 扩展或 CLI）
- 系统通知已开启：设置 → 系统 → 通知和操作 → 开启通知
- Windows PowerShell 的通知未被屏蔽

---

## 快速安装

### 步骤 1：创建 hooks 目录

```powershell
mkdir "$env:USERPROFILE\.claude\hooks" -Force
```

### 步骤 2：创建三个 PowerShell 脚本

将下文的三个脚本保存到 `~/.claude/hooks/` 目录。

### 步骤 3：创建配置文件

将 `claude-notifier-config.json` 保存到同一目录。

### 步骤 4：配置 `settings.json`

将 hooks 注册信息写入 `~/.claude/settings.json`。

### 步骤 5：重启 Claude Code 会话

修改完成后，必须**关闭并重新打开** Claude Code 会话，hook 才会生效。

---

## 完整文件清单

```
C:\Users\<用户名>\.claude\
├── settings.json                          ← hooks 注册 + 权限白名单
└── hooks\
    ├── claude-notifier-on-stop.ps1        ← 任务完成通知
    ├── claude-notifier-on-permission.ps1  ← 需要授权通知
    ├── claude-notifier-on-question.ps1    ← 提问通知
    └── claude-notifier-config.json        ← 通知偏好配置
```

---

## 文件 1：`settings.json`

路径：`C:\Users\<用户名>\.claude\settings.json`

> 如果你已有此文件，只需合并 `hooks` 部分。`permissions` 部分为可选的工具白名单。

```json
{
  "permissions": {
    "allow": [
      "Read",
      "Edit",
      "Write",
      "Glob",
      "Grep",
      "WebFetch",
      "WebSearch",
      "Bash(ls*)",
      "Bash(pwd*)",
      "Bash(cat*)",
      "Bash(head*)",
      "Bash(tail*)",
      "Bash(wc*)",
      "Bash(echo*)",
      "Bash(which*)",
      "Bash(find*)",
      "Bash(grep*)",
      "Bash(cd*)",
      "Bash(mkdir*)",
      "Bash(cp*)",
      "Bash(mv*)",
      "Bash(touch*)",
      "Bash(diff*)",
      "Bash(sort*)",
      "Bash(uniq*)",
      "Bash(sed*)",
      "Bash(awk*)",
      "Bash(git status*)",
      "Bash(git log*)",
      "Bash(git diff*)",
      "Bash(git show*)",
      "Bash(git branch*)",
      "Bash(git add*)",
      "Bash(git commit*)",
      "Bash(git stash*)",
      "Bash(git tag*)",
      "Bash(git config*)",
      "Bash(git init*)",
      "Bash(git remote*)",
      "Bash(gh auth*)",
      "Bash(gh api*)",
      "Bash(python*)",
      "Bash(python3*)",
      "Bash(pip*)",
      "Bash(pdflatex*)",
      "Bash(powershell*)"
    ],
    "deny": [
      "Bash(rm -rf*)",
      "Bash(rm -r*)",
      "Bash(git push --force*)",
      "Bash(git push -f*)",
      "Bash(git reset --hard*)",
      "Bash(git clean*)"
    ]
  },
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -NonInteractive -ExecutionPolicy Bypass -File \"C:\\Users\\<用户名>\\.claude\\hooks\\claude-notifier-on-stop.ps1\""
          }
        ]
      }
    ],
    "PermissionRequest": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -NonInteractive -ExecutionPolicy Bypass -File \"C:\\Users\\<用户名>\\.claude\\hooks\\claude-notifier-on-permission.ps1\""
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "AskUserQuestion",
        "hooks": [
          {
            "type": "command",
            "command": "powershell -NoProfile -NonInteractive -ExecutionPolicy Bypass -File \"C:\\Users\\<用户名>\\.claude\\hooks\\claude-notifier-on-question.ps1\""
          }
        ]
      }
    ]
  }
}
```

**注意**：将所有 `<用户名>` 替换为你的 Windows 用户名。

### 工具白名单说明

| 配置项 | 效果 |
|---|---|
| `allow` 列表中的工具 | 自动执行，不弹授权框，也不触发 Permission 通知 |
| `deny` 列表中的工具 | 直接拒绝执行 |
| 不在两个列表中的工具 | 弹出授权对话框，同时触发 Permission 通知 |

通配符 `*` 匹配命令前缀，如 `Bash(git status*)` 匹配所有以 `git status` 开头的命令。

---

## 文件 2：`claude-notifier-on-stop.ps1`

路径：`~/.claude/hooks/claude-notifier-on-stop.ps1`

这是最重要的脚本——Claude 完成回复时触发。包含 UTF-8 编码修复和调试日志。

```powershell
# Claude Notifier - Stop hook (PowerShell, v2)
# Toast notification when Claude finishes responding.

# Debug log - BEFORE ErrorActionPreference, using .NET directly
$_logPath = [System.IO.Path]::Combine($env:USERPROFILE, '.claude', 'hooks', 'stop-debug.log')
[System.IO.File]::AppendAllText($_logPath, "$(Get-Date -Format 'yyyy-MM-dd HH:mm:ss.fff') - Hook invoked`r`n")

$ErrorActionPreference = 'SilentlyContinue'

$hooksDir = Join-Path ($env:USERPROFILE) '.claude' 'hooks'
$muteFlag = Join-Path $hooksDir 'claude-notifier-muted'
$configFile = Join-Path $hooksDir 'claude-notifier-config.json'

$winSounds = @{
    'Windows Notify' = 'C:\Windows\Media\Windows Notify.wav'
    'tada'           = 'C:\Windows\Media\tada.wav'
    'chimes'         = 'C:\Windows\Media\chimes.wav'
    'chord'          = 'C:\Windows\Media\chord.wav'
    'ding'           = 'C:\Windows\Media\ding.wav'
    'notify'         = 'C:\Windows\Media\notify.wav'
    'ringin'         = 'C:\Windows\Media\ringin.wav'
    'Windows Background' = 'C:\Windows\Media\Windows Background.wav'
}

# Read stdin as UTF-8 to handle Chinese characters
$reader = New-Object System.IO.StreamReader([Console]::OpenStandardInput(), [System.Text.Encoding]::UTF8)
$raw = $reader.ReadToEnd()
$data = $null
try { $data = $raw | ConvertFrom-Json } catch {
    [System.IO.File]::AppendAllText($_logPath, "$(Get-Date -Format 'HH:mm:ss.fff') - JSON parse failed, will send default notification`r`n")
    # Don't exit - continue with default notification
}

if ($data -and $data.stop_hook_active) { exit 0 }
if (Test-Path $muteFlag) { exit 0 }

$reason = 'done'

if ($data) {
    $transcript = $data.transcript_path
    if ($transcript -and (Test-Path $transcript)) {
        try {
            $lines = Get-Content $transcript -Tail 20
            for ($i = $lines.Count - 1; $i -ge 0; $i--) {
                try {
                    $msg = $lines[$i] | ConvertFrom-Json
                    if ($msg.role -eq 'assistant' -and $msg.content -and $msg.content.Count -gt 0) {
                        $last = $msg.content[$msg.content.Count - 1]
                        if ($last.type -eq 'tool_use' -and $last.name -eq 'AskUserQuestion') {
                            $reason = 'question'
                        } elseif ($last.type -eq 'text' -and $last.text -and $last.text.Trim().EndsWith('?')) {
                            $reason = 'question'
                        }
                        break
                    }
                } catch {}
            }
        } catch {}
    }
}

# Read config
$config = $null
try { $config = (Get-Content $configFile -Raw) | ConvertFrom-Json } catch {}

$configKey = if ($reason -eq 'question') { 'asksQuestion' } else { 'taskCompleted' }
$eventCfg = if ($config -and $config.$configKey) { $config.$configKey } else { $null }
$level = if ($eventCfg -and $eventCfg.level) { $eventCfg.level } else { 'sound+popup' }

if ($level -eq 'off') { exit 0 }

$defaultSounds = @{ question = 'C:\Windows\Media\Windows Notify.wav'; done = 'C:\Windows\Media\tada.wav' }
$soundName = if ($eventCfg -and $eventCfg.sound) { $eventCfg.sound } else { '' }
$soundPath = if ($winSounds.ContainsKey($soundName)) { $winSounds[$soundName] } else { $defaultSounds[$reason] }

$messages = @{ question = 'Claude is asking you a question.'; done = 'Claude has finished the task.' }

# Play sound
if ($level -eq 'sound+popup' -or $level -eq 'sound') {
    try {
        if (Test-Path $soundPath) { (New-Object Media.SoundPlayer $soundPath).PlaySync() }
        else { [console]::Beep(800, 300) }
    } catch {}
}

# OS notification (Toast)
if ($level -eq 'sound+popup' -or $level -eq 'popup') {
    try {
        [Windows.UI.Notifications.ToastNotificationManager, Windows.UI.Notifications, ContentType = WindowsRuntime] | Out-Null
        [Windows.Data.Xml.Dom.XmlDocument, Windows.Data.Xml.Dom, ContentType = WindowsRuntime] | Out-Null
        $toastXml = @"
<toast>
  <visual>
    <binding template="ToastGeneric">
      <text>Claude Notifier</text>
      <text>$($messages[$reason])</text>
    </binding>
  </visual>
</toast>
"@
        $xml = New-Object Windows.Data.Xml.Dom.XmlDocument
        $xml.LoadXml($toastXml)
        $toast = [Windows.UI.Notifications.ToastNotification]::new($xml)
        $toast.Tag = "claude-$(Get-Date -UFormat %s)"
        $toast.Group = "claude-notifier"
        [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier("{1AC14E77-02E7-4E5D-B744-2EB1AE5198B7}\WindowsPowerShell\v1.0\powershell.exe").Show($toast)
        [System.IO.File]::AppendAllText($_logPath, "$(Get-Date -Format 'HH:mm:ss.fff') - Toast sent OK (reason=$reason)`r`n")
    } catch {
        [System.IO.File]::AppendAllText($_logPath, "$(Get-Date -Format 'HH:mm:ss.fff') - Toast FAILED: $($_.Exception.Message)`r`n")
    }
}

# Write signal file (can be used by other tools)
try {
    Set-Content -Path (Join-Path $hooksDir 'claude-signal') -Value "$reason $(Get-Date -UFormat %s)" -NoNewline
} catch {}
```

---

## 文件 3：`claude-notifier-on-permission.ps1`

路径：`~/.claude/hooks/claude-notifier-on-permission.ps1`

Claude 调用未在白名单中的工具时触发。

```powershell
# Claude Notifier - PermissionRequest hook (PowerShell, v2)
# Toast notification when Claude needs tool permission.
$ErrorActionPreference = 'SilentlyContinue'

$hooksDir = Join-Path ($env:USERPROFILE) '.claude' 'hooks'
$muteFlag = Join-Path $hooksDir 'claude-notifier-muted'
$configFile = Join-Path $hooksDir 'claude-notifier-config.json'

$winSounds = @{
    'Windows Notify' = 'C:\Windows\Media\Windows Notify.wav'
    'tada'           = 'C:\Windows\Media\tada.wav'
    'chimes'         = 'C:\Windows\Media\chimes.wav'
    'chord'          = 'C:\Windows\Media\chord.wav'
    'ding'           = 'C:\Windows\Media\ding.wav'
    'notify'         = 'C:\Windows\Media\notify.wav'
    'ringin'         = 'C:\Windows\Media\ringin.wav'
    'Windows Background' = 'C:\Windows\Media\Windows Background.wav'
}

$raw = [Console]::In.ReadToEnd()
try { $data = $raw | ConvertFrom-Json } catch { exit 0 }

if (Test-Path $muteFlag) { exit 0 }

# Read config
$config = $null
try { $config = (Get-Content $configFile -Raw) | ConvertFrom-Json } catch {}

$eventCfg = if ($config -and $config.needsPermission) { $config.needsPermission } else { $null }
$level = if ($eventCfg -and $eventCfg.level) { $eventCfg.level } else { 'sound+popup' }

if ($level -eq 'off') { exit 0 }

$soundName = if ($eventCfg -and $eventCfg.sound) { $eventCfg.sound } else { '' }
$soundPath = if ($winSounds.ContainsKey($soundName)) { $winSounds[$soundName] } else { 'C:\Windows\Media\Windows Notify.wav' }

# Play sound
if ($level -eq 'sound+popup' -or $level -eq 'sound') {
    try {
        if (Test-Path $soundPath) { (New-Object Media.SoundPlayer $soundPath).PlaySync() }
        else { [console]::Beep(800, 300) }
    } catch {}
}

# OS notification (Toast)
if ($level -eq 'sound+popup' -or $level -eq 'popup') {
    try {
        $tool = if ($data.tool_name) { $data.tool_name } else { 'a tool' }
        $message = "Claude needs permission to use $tool."
        [Windows.UI.Notifications.ToastNotificationManager, Windows.UI.Notifications, ContentType = WindowsRuntime] | Out-Null
        [Windows.Data.Xml.Dom.XmlDocument, Windows.Data.Xml.Dom, ContentType = WindowsRuntime] | Out-Null
        $toastXml = @"
<toast>
  <visual>
    <binding template="ToastGeneric">
      <text>Claude Notifier</text>
      <text>$message</text>
    </binding>
  </visual>
</toast>
"@
        $xml = New-Object Windows.Data.Xml.Dom.XmlDocument
        $xml.LoadXml($toastXml)
        $toast = [Windows.UI.Notifications.ToastNotification]::new($xml)
        [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier("{1AC14E77-02E7-4E5D-B744-2EB1AE5198B7}\WindowsPowerShell\v1.0\powershell.exe").Show($toast)
    } catch {}
}

# Write signal file
try {
    Set-Content -Path (Join-Path $hooksDir 'claude-signal') -Value "input $(Get-Date -UFormat %s)" -NoNewline
} catch {}
```

---

## 文件 4：`claude-notifier-on-question.ps1`

路径：`~/.claude/hooks/claude-notifier-on-question.ps1`

Claude 调用 `AskUserQuestion` 工具时触发（目前在 VSCode 中不稳定，见 [已知问题](#已知问题)）。

```powershell
# Claude Notifier - PreToolUse hook for AskUserQuestion (PowerShell, v2)
# Toast notification when Claude asks the user a question.
$ErrorActionPreference = 'SilentlyContinue'

$hooksDir = Join-Path ($env:USERPROFILE) '.claude' 'hooks'
$muteFlag = Join-Path $hooksDir 'claude-notifier-muted'
$configFile = Join-Path $hooksDir 'claude-notifier-config.json'

$winSounds = @{
    'Windows Notify' = 'C:\Windows\Media\Windows Notify.wav'
    'tada'           = 'C:\Windows\Media\tada.wav'
    'chimes'         = 'C:\Windows\Media\chimes.wav'
    'chord'          = 'C:\Windows\Media\chord.wav'
    'ding'           = 'C:\Windows\Media\ding.wav'
    'notify'         = 'C:\Windows\Media\notify.wav'
    'ringin'         = 'C:\Windows\Media\ringin.wav'
    'Windows Background' = 'C:\Windows\Media\Windows Background.wav'
}

$raw = [Console]::In.ReadToEnd()
try { $data = $raw | ConvertFrom-Json } catch { exit 0 }

if (Test-Path $muteFlag) { exit 0 }

# Read config
$config = $null
try { $config = (Get-Content $configFile -Raw) | ConvertFrom-Json } catch {}

$eventCfg = if ($config -and $config.asksQuestion) { $config.asksQuestion } else { $null }
$level = if ($eventCfg -and $eventCfg.level) { $eventCfg.level } else { 'sound+popup' }

if ($level -eq 'off') { exit 0 }

$soundName = if ($eventCfg -and $eventCfg.sound) { $eventCfg.sound } else { '' }
$soundPath = if ($winSounds.ContainsKey($soundName)) { $winSounds[$soundName] } else { 'C:\Windows\Media\Windows Notify.wav' }

# Play sound
if ($level -eq 'sound+popup' -or $level -eq 'sound') {
    try {
        if (Test-Path $soundPath) { (New-Object Media.SoundPlayer $soundPath).PlaySync() }
        else { [console]::Beep(800, 300) }
    } catch {}
}

# OS notification (Toast)
if ($level -eq 'sound+popup' -or $level -eq 'popup') {
    try {
        [Windows.UI.Notifications.ToastNotificationManager, Windows.UI.Notifications, ContentType = WindowsRuntime] | Out-Null
        [Windows.Data.Xml.Dom.XmlDocument, Windows.Data.Xml.Dom, ContentType = WindowsRuntime] | Out-Null
        $toastXml = @"
<toast>
  <visual>
    <binding template="ToastGeneric">
      <text>Claude Notifier</text>
      <text>Claude is asking you a question.</text>
    </binding>
  </visual>
</toast>
"@
        $xml = New-Object Windows.Data.Xml.Dom.XmlDocument
        $xml.LoadXml($toastXml)
        $toast = [Windows.UI.Notifications.ToastNotification]::new($xml)
        [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier("{1AC14E77-02E7-4E5D-B744-2EB1AE5198B7}\WindowsPowerShell\v1.0\powershell.exe").Show($toast)
    } catch {}
}

# Write signal file
try {
    Set-Content -Path (Join-Path $hooksDir 'claude-signal') -Value "question $(Get-Date -UFormat %s)" -NoNewline
} catch {}
```

---

## 文件 5：`claude-notifier-config.json`

路径：`~/.claude/hooks/claude-notifier-config.json`

```json
{
  "taskCompleted": {
    "level": "popup",
    "sound": "tada"
  },
  "needsPermission": {
    "level": "popup",
    "sound": "Windows Notify"
  },
  "asksQuestion": {
    "level": "popup",
    "sound": "Windows Notify"
  }
}
```

### `level` 可选值

| 值 | 效果 |
|---|---|
| `"sound+popup"` | 声音 + Toast 弹窗 |
| `"popup"` | 仅 Toast 弹窗（推荐） |
| `"sound"` | 仅声音 |
| `"off"` | 关闭该事件的通知 |

### 可用声音名称

`tada`, `chimes`, `chord`, `ding`, `notify`, `ringin`, `Windows Notify`, `Windows Background`

（对应 `C:\Windows\Media\` 下的 .wav 文件）

---

## 测试

### 手动测试 Toast 弹窗

打开 PowerShell，运行：

```powershell
echo '{"stop_hook_active": false}' | powershell -NoProfile -NonInteractive -ExecutionPolicy Bypass -File "$env:USERPROFILE\.claude\hooks\claude-notifier-on-stop.ps1"
```

如果右下角弹出 Toast，说明配置正确。

### 查看调试日志

Stop hook 会写调试日志：

```powershell
Get-Content "$env:USERPROFILE\.claude\hooks\stop-debug.log" -Tail 10
```

日志记录每次 hook 调用时间和 Toast 发送结果。

### 静音控制

```powershell
# 静音所有通知
New-Item "$env:USERPROFILE\.claude\hooks\claude-notifier-muted" -ItemType File

# 恢复通知
Remove-Item "$env:USERPROFILE\.claude\hooks\claude-notifier-muted"
```

---

## 技术细节

### 为什么不能用 `NotifyIcon.ShowBalloonTip`

这是 Windows Forms 的气泡通知方式。它需要 Windows 消息循环（message loop）才能渲染气泡，但 Claude Code 调用 hook 时使用的是**非交互式 PowerShell**（`-NonInteractive`），没有消息循环。结果：脚本执行无报错，但用户什么都看不到。

### 为什么用 `ToastNotificationManager`

Windows 10 的 Toast 通知由系统通知服务独立处理，不依赖调用进程的消息循环。即使 PowerShell 进程立即退出，Toast 仍然会显示。

### `AppUserModelID` 的坑

`CreateToastNotifier()` 需要传入一个**已在系统注册的 AppUserModelID**。如果传入自定义字符串（如 `"Claude Notifier"`），Toast 会静默失败。

必须使用 PowerShell 的已注册 ID：

```
{1AC14E77-02E7-4E5D-B744-2EB1AE5198B7}\WindowsPowerShell\v1.0\powershell.exe
```

副作用：弹窗的应用图标显示为 "Windows PowerShell"。

### UTF-8 编码问题（关键修复）

**问题**：Windows PowerShell 5.1 默认用系统编码（中文系统为 GBK/GB2312）读取 stdin。Claude Code 通过 stdin 传入的 JSON 是 UTF-8 编码的，且 `last_assistant_message` 字段可能包含中文。编码不匹配导致 `ConvertFrom-Json` 解析失败。

**表现**：Claude 用英文回复时通知正常，用中文回复时通知丢失，时有时无。

**修复**：在 `on-stop.ps1` 中用 .NET `StreamReader` 强制 UTF-8 读取：

```powershell
# 替换原来的 $raw = [Console]::In.ReadToEnd()
$reader = New-Object System.IO.StreamReader(
    [Console]::OpenStandardInput(),
    [System.Text.Encoding]::UTF8
)
$raw = $reader.ReadToEnd()
```

同时，JSON 解析失败时不退出，继续发送默认通知：

```powershell
$data = $null
try { $data = $raw | ConvertFrom-Json } catch {
    # 解析失败 → 继续发默认通知，不静默退出
}
```

### Toast 去重

Windows 会对相同 Tag + Group 的 Toast 做去重。为避免连续通知被吞掉，每条 Toast 使用唯一 Tag：

```powershell
$toast.Tag = "claude-$(Get-Date -UFormat %s)"
$toast.Group = "claude-notifier"
```

---

## 已知问题

### PreToolUse (AskUserQuestion) 通知不稳定

`PreToolUse` hook 在 VSCode 扩展环境中行为不稳定：

1. 全局 `~/.claude/settings.json` 中的 PreToolUse hook 在 VSCode 中不被调用
2. 项目级 `.claude/settings.local.json` 中的 PreToolUse hook 曾触发过一次，后续不再触发
3. 可能与 VSCode 扩展的 hook 加载机制有关

**替代方案**：Stop hook 在 Claude 提问并等待回答时也会触发，显示 "Claude has finished the task."。虽然消息不够精确，但功能上可以替代。

**TODO**：
- [ ] 关注 Claude Code 更新，检查 PreToolUse hook 是否修复
- [ ] 修复后对 `on-question.ps1` 和 `on-permission.ps1` 应用同样的 UTF-8 stdin 修复

### macOS 支持

- [ ] 实现 macOS 版本通知脚本（使用 `osascript` / `terminal-notifier`）
- [ ] 编写 macOS 安装指南
- [ ] 测试 macOS 上三种 hook 的触发行为

---

## 问题排查

| 问题 | 原因 | 解决 |
|---|---|---|
| 手动测试有弹窗，实际没有 | Hook 未被加载 | 重启 Claude Code 会话 |
| 脚本无报错但没弹窗 | 用了 NotifyIcon 而非 Toast | 检查是否使用 ToastNotificationManager |
| Toast 代码正确但没弹窗 | AppUserModelID 错误 | 使用 PowerShell 的已注册 ID |
| 通知时有时无 | stdin 中文编码问题 | 用 StreamReader + UTF-8 读取 stdin |
| Windows 通知被屏蔽 | 系统设置 | 设置 → 通知 → 确保 PowerShell 通知已开启 |
| 连续通知只收到一条 | Toast 去重 | 确保每条 Toast 使用唯一 Tag |

---

## 致谢

基于 [Claude Notifier](https://marketplace.visualstudio.com/items?itemName=SingularityInc.claude-notifier) 插件的 hook 设计思路，重写为独立的 Windows Toast 通知实现。

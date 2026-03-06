# Claude Notifier 插件 Windows 配置指南

## 背景

在使用 Claude Code（VSCode 扩展）时，当 Claude 完成任务、需要授权或提问时，用户可能已经切到其他窗口。Claude Notifier 插件通过系统弹窗通知用户回来操作。

## 使用的插件

- **插件名称**: Claude Notifier
- **插件 ID**: `singularityinc.claude-notifier`
- **版本**: 2.1.0
- **安装方式**: VSCode 扩展商店搜索 "Claude Notifier"，或命令行 `code --install-extension SingularityInc.claude-notifier`
- **插件功能**: 注册三种 Claude Code hook，在特定事件发生时播放声音和/或弹出系统通知

## 插件默认行为（不生效的原因）

插件安装后会自动在 `~/.claude/settings.json` 中注册三种 hook，并在 `~/.claude/hooks/` 目录下生成对应的 PowerShell 脚本。

### 默认使用的通知方式

插件默认使用 `NotifyIcon.ShowBalloonTip`（Windows 气泡通知）来弹窗：

```powershell
Add-Type -AssemblyName System.Windows.Forms
$n = New-Object System.Windows.Forms.NotifyIcon
$n.Icon = [System.Drawing.SystemIcons]::Information
$n.Visible = $true
$n.ShowBalloonTip(3000, 'Claude Notifier', $message, [System.Windows.Forms.ToolTipIcon]::None)
Start-Sleep -Milliseconds 500
$n.Dispose()
```

### 为什么这种方式在 Windows 10 上不生效

`NotifyIcon.ShowBalloonTip` 需要 Windows 消息循环（message loop）才能正确渲染气泡。Claude Code 调用 hook 脚本时使用的是**非交互式 PowerShell 进程**（`-NonInteractive` 参数），这种进程没有消息循环。结果就是：

- PowerShell 进程启动
- 创建 NotifyIcon 对象
- 调用 ShowBalloonTip
- 气泡来不及渲染，进程就结束了（或 `Start-Sleep` 时间不足以让消息循环处理事件）

所以脚本执行无报错，但用户什么都看不到。

### 第二个坑：中文编码导致通知时有时无

即使替换为 Toast 通知后，仍可能出现**通知时有时无**的现象。根本原因：

**Windows PowerShell 5.1 默认用系统编码（GBK/GB2312）读取 stdin，而 Claude Code 通过 stdin 传入的 JSON 是 UTF-8 编码的。** 当 `last_assistant_message` 字段包含中文时，编码不匹配导致 `ConvertFrom-Json` 解析失败，脚本在 `catch { exit 0 }` 处静默退出——不发通知也不报错。

表现：
- Claude 用英文回复时 → 通知正常
- Claude 用中文回复时 → 通知丢失
- 混合使用时 → 时有时无

**修复方案**：用 .NET 的 `StreamReader` 强制 UTF-8 读取 stdin，并且 JSON 解析失败时不退出而是继续发默认通知：

```powershell
# 替换原来的 $raw = [Console]::In.ReadToEnd()
$reader = New-Object System.IO.StreamReader([Console]::OpenStandardInput(), [System.Text.Encoding]::UTF8)
$raw = $reader.ReadToEnd()
$data = $null
try { $data = $raw | ConvertFrom-Json } catch {
    # JSON 解析失败时不退出，继续发默认通知
}
```

同时，后续所有引用 `$data` 的地方需要加 `if ($data)` 保护。

## 解决方案：改用 Windows 10 Toast 通知

### 核心修改

将 `NotifyIcon.ShowBalloonTip` 替换为 Windows 10 原生的 **Toast 通知 API**（`ToastNotificationManager`）。Toast 通知由 Windows 通知服务独立处理，不依赖调用进程的消息循环。

### 关键代码

```powershell
# 加载 WinRT 类型
[Windows.UI.Notifications.ToastNotificationManager, Windows.UI.Notifications, ContentType = WindowsRuntime] | Out-Null
[Windows.Data.Xml.Dom.XmlDocument, Windows.Data.Xml.Dom, ContentType = WindowsRuntime] | Out-Null

# 定义 Toast XML
$toastXml = @"
<toast>
  <visual>
    <binding template="ToastGeneric">
      <text>Claude Notifier</text>
      <text>通知内容</text>
    </binding>
  </visual>
</toast>
"@

# 创建并显示 Toast
$xml = New-Object Windows.Data.Xml.Dom.XmlDocument
$xml.LoadXml($toastXml)
$toast = [Windows.UI.Notifications.ToastNotification]::new($xml)
[Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier(
    "{1AC14E77-02E7-4E5D-B744-2EB1AE5198B7}\WindowsPowerShell\v1.0\powershell.exe"
).Show($toast)
```

### 为什么必须使用 PowerShell 的 AppUserModelID

`CreateToastNotifier()` 需要传入一个**已在系统注册的 AppUserModelID**。如果传入自定义字符串（如 `"Claude Notifier"`），Toast 会静默失败——不报错，但不弹窗。

PowerShell 的已注册 ID 是：
```
{1AC14E77-02E7-4E5D-B744-2EB1AE5198B7}\WindowsPowerShell\v1.0\powershell.exe
```

因此弹窗标题显示为 "Windows PowerShell"，内容标题显示 "Claude Notifier"。

## 需要修改的文件

共三个 PowerShell 脚本，位于 `C:\Users\<用户名>\.claude\hooks\`：

### 1. `claude-notifier-on-stop.ps1`（任务完成通知）

找到原始的 `# OS notification` 代码块，将 `NotifyIcon.ShowBalloonTip` 部分替换为：

```powershell
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
        [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier("{1AC14E77-02E7-4E5D-B744-2EB1AE5198B7}\WindowsPowerShell\v1.0\powershell.exe").Show($toast)
    } catch {}
}
```

### 2. `claude-notifier-on-permission.ps1`（需要授权通知）

同样替换 `# OS notification` 代码块：

```powershell
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
```

### 3. `claude-notifier-on-question.ps1`（提问通知）

同样替换 `# OS notification` 代码块：

```powershell
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
```

## 配置（只要弹窗不要声音）

编辑 `~/.claude/hooks/claude-notifier-config.json`：

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

`level` 可选值：
- `"sound+popup"` — 声音 + 弹窗
- `"sound"` — 仅声音
- `"popup"` — 仅弹窗
- `"off"` — 关闭

## 三种通知的工作状态

| Hook 类型 | 事件 | 通知内容 | 状态 |
|---|---|---|---|
| **Stop** | Claude 完成回复 | "Claude has finished the task." | 正常工作 |
| **PermissionRequest** | Claude 需要工具授权 | "Claude needs permission to use Bash." | 正常工作 |
| **PreToolUse (AskUserQuestion)** | Claude 向用户提问 | "Claude is asking you a question." | 不稳定（见下文） |

### Stop 通知触发时机

- Claude 完成回复、停止响应时触发
- 无论你在 VSCode 还是其他窗口，都会弹出 Toast
- 这是最常用的通知——你把任务交给 Claude 后去做别的事，完成时会提醒你

### Permission 通知触发时机

- Claude 调用一个**未在白名单中**的工具时触发
- 例如：白名单里有 `Bash(git status*)`，但 Claude 执行 `curl` 命令时就会触发
- 弹窗同时，VSCode 内也会显示授权对话框

### Question 通知（PreToolUse）的已知问题

`PreToolUse` hook 在 VSCode 扩展环境中行为不稳定：

1. 全局 `~/.claude/settings.json` 中的 PreToolUse hook 在 VSCode 中**不被调用**
2. 项目级 `.claude/settings.local.json` 中的 PreToolUse hook 曾触发过一次，但后续不再触发
3. 可能与 VSCode 扩展的 hook 加载机制有关（已知 bug："Found 0 hook matchers"）
4. **Stop hook 可以部分替代**：当 Claude 提问并等待回答时，Stop hook 也会触发，显示 "Claude has finished the task."

## Windows 用户完整配置步骤

### 前置条件

- Windows 10 或更高版本
- VSCode 已安装
- Claude Code VSCode 扩展已安装
- 系统通知已开启（设置 → 系统 → 通知和操作 → 开启通知）
- Windows PowerShell 通知未被屏蔽

### 步骤 1：安装插件

```
code --install-extension SingularityInc.claude-notifier
```

安装后**重启 VSCode**，插件会自动配置 hooks 和脚本。

### 步骤 2：验证自动生成的文件

确认以下文件存在：

```
C:\Users\<用户名>\.claude\hooks\claude-notifier-on-stop.ps1
C:\Users\<用户名>\.claude\hooks\claude-notifier-on-permission.ps1
C:\Users\<用户名>\.claude\hooks\claude-notifier-on-question.ps1
C:\Users\<用户名>\.claude\hooks\claude-notifier-config.json
```

确认 `~/.claude/settings.json` 的 `hooks` 部分包含 Stop、PermissionRequest、PreToolUse 三种配置。

### 步骤 3：修改三个 .ps1 脚本

按上文"需要修改的文件"部分，将每个脚本中的 `NotifyIcon.ShowBalloonTip` 替换为 Toast 通知代码。

**关键点**：只替换 `# OS notification` 代码块，不要改动脚本其他部分（stdin 读取、JSON 解析、config 读取等）。

### 步骤 4：配置通知偏好（可选）

编辑 `~/.claude/hooks/claude-notifier-config.json`，设置每种事件的 level 和 sound。

### 步骤 5：手动测试

打开 PowerShell，执行以下命令测试 Toast 是否正常：

```powershell
# 测试 Stop 通知
echo '{"stop_hook_active": false}' | powershell -NoProfile -NonInteractive -ExecutionPolicy Bypass -File "$env:USERPROFILE\.claude\hooks\claude-notifier-on-stop.ps1"

# 测试 Permission 通知
echo '{"tool_name": "Bash"}' | powershell -NoProfile -NonInteractive -ExecutionPolicy Bypass -File "$env:USERPROFILE\.claude\hooks\claude-notifier-on-permission.ps1"
```

如果右下角弹出 Toast 通知，说明配置成功。

### 步骤 6：重启 Claude Code 会话

修改完脚本后，**必须关闭并重新打开 Claude Code 会话**，hook 配置才会生效。

## 注意事项

1. **不要在脚本中重复读取 stdin**：`[Console]::In.ReadToEnd()` 只能调用一次。如果调用两次，第二次会得到空字符串，导致 JSON 解析失败，整个脚本静默退出。

2. **AppUserModelID 必须是已注册的**：自定义字符串会导致 Toast 静默失败。使用 PowerShell 的已注册 ID。

3. **插件更新可能覆盖修改**：Claude Notifier 更新时可能重新生成 .ps1 脚本，覆盖你的 Toast 修改。更新后需要重新应用修改。

4. **VSCode 全屏时通知可能不显示**：Toast 通知在 Windows 全屏应用下可能被压到通知中心。切到其他窗口或非全屏模式时会正常弹出。

5. **静音控制**：创建文件 `~/.claude/hooks/claude-notifier-muted` 可静音所有通知，删除该文件恢复。

## 问题排查

| 问题 | 可能原因 | 解决方法 |
|---|---|---|
| 手动测试有弹窗，实际使用没有 | Hook 没有被 Claude Code 调用 | 重启 Claude Code 会话 |
| 脚本执行无报错但没弹窗 | 还在用 NotifyIcon 而非 Toast | 检查脚本是否已替换为 Toast 代码 |
| Toast 代码正确但没弹窗 | AppUserModelID 错误 | 使用 PowerShell 的已注册 ID |
| Permission 有弹窗但 Stop 没有 | stdin 被读取了两次 | 确保脚本中只有一次 `ReadToEnd()` |
| **通知时有时无** | **stdin 中文编码问题（GBK vs UTF-8）** | **用 `StreamReader` + UTF-8 读取 stdin，解析失败不退出** |
| Windows 通知被屏蔽 | 系统设置问题 | 设置 → 通知 → 确保 PowerShell 通知已开启 |

## 附：工具白名单配置

在 `~/.claude/settings.json` 的 `permissions` 部分，可以配置自动允许和禁止的工具，避免频繁弹出授权对话框：

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
  }
}
```

**说明**：
- `allow` 中的工具会自动执行，不弹授权框
- `deny` 中的工具会被直接拒绝
- 不在两个列表中的工具会触发 Permission 通知，等待用户授权
- 使用通配符 `*` 匹配命令前缀（如 `Bash(git status*)` 匹配所有以 `git status` 开头的命令）

## 总结

Claude Notifier 插件在 macOS 上开箱即用，但在 Windows 上需要手动将通知方式从 `NotifyIcon.ShowBalloonTip` 改为 `ToastNotificationManager`。修改后 Stop（任务完成）和 Permission（需要授权）两种通知可以稳定工作，足以覆盖日常使用场景。

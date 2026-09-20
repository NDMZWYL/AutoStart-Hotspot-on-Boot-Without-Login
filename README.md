# AutoStart-Hotspot-on-Boot-Without-Login   自动启动热点-无需登录

> 在 Windows 10/11 系统启动时自动开启移动热点，无需用户登录，无需手动操作。

## 项目简介

本方案通过 **PowerShell 脚本** 调用 Windows 运行时（WinRT）的 `NetworkOperatorTetheringManager` API，实现移动热点的程序化开启；再通过 **Windows 任务计划程序** 将脚本配置为以 SYSTEM 账户在系统启动后自动执行，从而实现无需用户登录即可自动开启移动热点。

### 核心特性

| 特性 | 说明 |
|------|------|
| **无需登录** | 系统启动后即执行，不依赖用户登录会话 |
| **静默运行** | 后台执行，无窗口闪现 |
| **自动重试** | 任务失败后自动重试，应对网络服务初始化延迟 |
| **日志记录** | 完整记录执行过程，便于排查问题 |
| **防重复启动** | 检测热点状态，已开启时不重复操作 |

### 方案对比

| 方案 | 登录前可用 | 稳定性 | 配置复杂度 |
|------|-----------|--------|-----------|
| 启动文件夹 | ❌ | 中 | 低 |
| 任务计划（登录触发） | ❌ | 中 | 低 |
| **本方案（SYSTEM + 启动触发）** | ✅ | 高 | 中 |


## 工作原理

### 执行链路

```
系统启动 → 任务计划程序触发 → 以 SYSTEM 身份运行 PowerShell
    → 加载 WinRT 程序集 → 获取网络连接配置文件
    → 创建热点管理器 → 检查热点状态 → 启动热点
```

### 关键技术

**WinRT 异步同步化**：PowerShell 原生不支持 C# 的 `await` 关键字。脚本通过 `System.Runtime.WindowsRuntime` 程序集中的 `AsTask` 扩展方法，将 WinRT 异步操作（`IAsyncOperation<T>`）转换为 .NET 的 `Task<T>`，再通过 `Task.Wait()` 阻塞等待完成，最后通过 `Task.Result` 获取返回值。

**热点管理器**：`NetworkOperatorTetheringManager` 是 WinRT 提供的热点管理类，通过 `CreateFromConnectionProfile()` 静态方法创建实例，使用指定的网络连接配置文件作为公共接口，Wi-Fi 作为专用接口。

### 关键 API

| API | 用途 |
|-----|------|
| `NetworkInformation.GetInternetConnectionProfile()` | 获取当前互联网连接配置文件 |
| `NetworkOperatorTetheringManager.CreateFromConnectionProfile()` | 创建热点管理器实例 |
| `TetheringOperationalState` | 获取热点运行状态（`1` = 已开启） |
| `StartTetheringAsync()` | 异步启动热点 |
| `NetworkOperatorTetheringOperationResult.Status` | 获取操作结果状态码（`0` = 成功） |


## 快速开始

### 步骤 1：手动配置热点

按 `Win + I` 打开设置 → 网络和 Internet → 移动热点，先完整配置一次：

- 设置 SSID（热点名称）
- 设置至少 8 位密码
- 选择共享源（如以太网）
- 频段建议选 **2.4GHz**（兼容性更佳）
- **关闭“节能”选项**（否则无设备连接 5 分钟后热点会自动关闭）

手动开启热点，确认手机等设备能正常搜索并连接。

### 步骤 2：设置 PowerShell 执行策略

以管理员身份打开终端，执行：

```powershell
Set-ExecutionPolicy RemoteSigned
```

提示时输入 `A` 确认。此步骤不可跳过。

### 步骤 3：部署脚本

将 `StartHotspot.ps1` 复制到 `C:\Scripts\` 目录。

### 步骤 4：创建任务计划

```cmd
schtasks /create /TN "AutoStartHotspot" /RU SYSTEM /SC ONSTART /DELAY 0000:30 /TR "powershell.exe -ExecutionPolicy Bypass -NoProfile -WindowStyle Hidden -File C:\Scripts\StartHotspot.ps1" /RL HIGHEST /F
```

或按照 [任务计划程序配置](#任务计划程序配置) 手动创建。

### 步骤 5：验证

重启计算机，停留在登录界面，用另一台设备搜索热点 SSID，确认能够连接。


## 脚本详解

### 完整脚本

```powershell
<#
.SYNOPSIS
    自动启动 Windows 移动热点（无需用户登录）。
.DESCRIPTION
    通过 WinRT API 程序化开启移动热点，适用于任务计划程序在系统启动时
    以 SYSTEM 账户自动执行。包含错误处理、状态检查和日志记录。
.NOTES
    需在任务计划程序中以 SYSTEM 账户、最高权限运行。
#>

# ==================== 配置区 ====================
$LogDir  = "$env:ProgramData\AutoHotspot"
$LogFile = "$LogDir\hotspot.log"
# ================================================

# ---------- 日志函数 ----------
function Write-Log {
    param([string]$Message, [string]$Level = "INFO")
    $timestamp = Get-Date -Format "yyyy-MM-dd HH:mm:ss"
    $entry = "[$timestamp] [$Level] $Message"
    if (-not (Test-Path $LogDir)) {
        New-Item -Path $LogDir -ItemType Directory -Force | Out-Null
    }
    Add-Content -Path $LogFile -Value $entry -Encoding UTF8
}

# ---------- 主逻辑 ----------
try {
    Write-Log "===== 自动热点脚本开始执行 ====="

    # 1. 加载 WinRT 程序集
    Add-Type -AssemblyName System.Runtime.WindowsRuntime
    Write-Log "已加载 System.Runtime.WindowsRuntime 程序集"

    # 2. 构建 Await 函数
    $asTaskGeneric = ([System.WindowsRuntimeSystemExtensions].GetMethods() | Where-Object {
        $_.Name -eq 'AsTask' -and
        $_.GetParameters().Count -eq 1 -and
        $_.GetParameters()[0].ParameterType.Name -eq 'IAsyncOperation`1'
    })[0]

    if (-not $asTaskGeneric) {
        throw "未能找到 AsTask 泛型方法，WinRT 程序集可能未正确加载。"
    }

    Function Await {
        param($WinRtTask, $ResultType)
        $asTask = $asTaskGeneric.MakeGenericMethod($ResultType)
        $netTask = $asTask.Invoke($null, @($WinRtTask))
        $netTask.Wait(-1) | Out-Null
        $netTask.Result
    }

    # 3. 获取互联网连接配置文件
    $connectionProfile = [Windows.Networking.Connectivity.NetworkInformation, Windows.Networking.Connectivity, ContentType=WindowsRuntime]::GetInternetConnectionProfile()

    if ($null -eq $connectionProfile) {
        throw "未检测到可用的互联网连接配置文件。请确认网络适配器已就绪。"
    }
    Write-Log "已获取连接配置文件：$($connectionProfile.ProfileName)"

    # 4. 创建热点管理器
    $tetheringManager = [Windows.Networking.NetworkOperators.NetworkOperatorTetheringManager, Windows.Networking.NetworkOperators, ContentType=WindowsRuntime]::CreateFromConnectionProfile($connectionProfile)

    if ($null -eq $tetheringManager) {
        throw "创建热点管理器失败。请确认无线网卡驱动支持承载网络。"
    }
    Write-Log "热点管理器创建成功"

    # 5. 检查当前热点状态并启动
    $currentState = $tetheringManager.TetheringOperationalState
    Write-Log "当前热点运行状态：$currentState"

    if ($currentState -eq 1) {
        Write-Log "热点已处于开启状态，无需重复启动。" "WARN"
    }
    else {
        $result = Await ($tetheringManager.StartTetheringAsync()) ([Windows.Networking.NetworkOperators.NetworkOperatorTetheringOperationResult])

        if ($result.Status -eq 0) {
            Write-Log "热点启动成功！状态码：$($result.Status)"
        }
        else {
            Write-Log "热点启动返回异常状态码：$($result.Status)，附加信息：$($result.AdditionalErrorMessage)" "ERROR"
        }
    }

    Write-Log "===== 脚本执行完毕 ====="
}
catch {
    Write-Log "脚本执行发生异常：$($_.Exception.Message)" "ERROR"
    Write-Log "异常堆栈：$($_.ScriptStackTrace)" "ERROR"
    exit 1
}
```

### 模块说明

| 模块 | 功能 | 关键技术 |
|------|------|----------|
| 异步等待函数 `Await` | 将 WinRT 异步操作同步化 | 反射调用 `AsTask` 泛型方法 |
| 获取连接配置文件 | 获取当前互联网连接 | `NetworkInformation.GetInternetConnectionProfile()` |
| 启动热点 | 创建管理器并启动共享 | `NetworkOperatorTetheringManager` |

### Await 函数原理

PowerShell 原生不支持 C# 的 `await` 关键字。`Await` 函数通过以下步骤实现异步同步化：

1. 通过反射查找 `WindowsRuntimeSystemExtensions` 类中签名匹配 `AsTask(IAsyncOperation<T>)` 的方法
2. 使用 `MakeGenericMethod` 将泛型方法实例化为具体返回类型
3. 调用 `Invoke` 执行转换，得到 .NET `Task` 对象
4. 通过 `Wait(-1)` 无限期等待任务完成

### 优化要点

| 优化点 | 说明 |
|--------|------|
| **日志记录** | 所有执行步骤写入 `$env:ProgramData\AutoHotspot\hotspot.log`，使用 `ProgramData` 目录确保 SYSTEM 账户有写入权限 |
| **空值检查** | 对 `connectionProfile` 和 `tetheringManager` 进行空值判断 |
| **异常捕获** | 使用 `try-catch` 包裹主逻辑，任何未处理异常都会被记录 |
| **返回状态检查** | 检查 `StartTetheringAsync()` 返回的 `Status` 属性 |
| **退出码** | 异常时 `exit 1`，便于任务计划程序判断执行结果 |


## 任务计划程序配置

### 通过命令行快速创建

```cmd
schtasks /create /TN "AutoStartHotspot" /RU SYSTEM /SC ONSTART /DELAY 0000:30 /TR "powershell.exe -ExecutionPolicy Bypass -NoProfile -WindowStyle Hidden -File C:\Scripts\StartHotspot.ps1" /RL HIGHEST /F
```

参数说明：

| 参数 | 说明 |
|------|------|
| `/RU SYSTEM` | 以 SYSTEM 账户运行 |
| `/SC ONSTART` | 系统启动时触发 |
| `/DELAY 0000:30` | 延迟 30 秒 |
| `/RL HIGHEST` | 最高权限 |
| `/F` | 强制覆盖同名任务 |

### 通过图形界面配置

#### 常规选项卡

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| **名称** | `AutoStartHotspot` | 建议使用英文 |
| **使用最高权限运行** | ✅ 勾选 | 热点启动需要管理员级权限 |
| **不管用户是否登录都要运行** | ✅ 勾选 | **核心设置**，确保登录前即可执行 |
| **不存储密码** | ✅ 勾选 | 使用 SYSTEM 账户时无需存储密码 |
| **配置为** | `Windows 11` / `Windows 10` | 选择与系统版本匹配的配置 |

> ⚠️ **关键提醒**：若在任务计划程序中无法勾选“不管用户是否登录都要运行”（选项灰色），请直接在用户账户框中输入 `SYSTEM`，保存任务后重新打开，该选项会自动选中。

#### 触发器选项卡

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| **开始任务** | `计算机启动时` | 系统启动后即触发 |
| **延迟任务时间** | `30 秒` 至 `1 分钟` | 给网络服务留出初始化时间 |
| **任务的运行时间超过此值则停止执行** | 取消勾选或设置较大值 | 避免任务被意外终止 |
| **已启用** | ✅ 勾选 | 确保触发器处于激活状态 |

关于延迟时间的说明：Windows 系统启动后，`WLAN AutoConfig`、`Internet Connection Sharing` 等网络服务需要一定时间完成初始化。若脚本执行过早，可能因网络适配器尚未就绪而失败。30 秒至 1 分钟的延迟是经过实践验证的合理值。

#### 操作选项卡

| 配置项 | 推荐值 |
|--------|--------|
| **操作** | `启动程序` |
| **程序或脚本** | `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe` |
| **添加参数** | `-ExecutionPolicy Bypass -NoProfile -WindowStyle Hidden -File "C:\Scripts\StartHotspot.ps1"` |
| **起始于** | `C:\Scripts` |

参数说明：

- `-ExecutionPolicy Bypass`：绕过执行策略限制，确保脚本可运行
- `-NoProfile`：不加载 PowerShell 配置文件，加快启动速度
- `-WindowStyle Hidden`：隐藏 PowerShell 窗口，实现后台静默执行
- `-File`：指定脚本完整路径

#### 条件选项卡

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| **只有在计算机使用交流电源时才启动** | ❌ 取消勾选 | 确保笔记本使用电池时也能执行 |
| **唤醒计算机运行此任务** | ❌ 取消勾选 | 避免意外唤醒导致耗电 |
| **只有在以下网络连接可用时才启动** | 不勾选 | 热点启动时网络可能尚未就绪 |

#### 设置选项卡

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| **允许按需运行任务** | ✅ 勾选 | 允许手动触发测试 |
| **如果任务失败，按以下频率重新启动** | ✅ 勾选，`1 分钟`，最多 `3` 次 | 网络服务未就绪时自动重试 |
| **如果任务运行时间超过以下时间，停止任务** | ✅ 勾选，`1 小时` | 避免脚本因异步等待卡死后无限运行 |
| **如果请求后任务还在运行，强行将其停止** | ✅ 勾选 | 确保重复触发时不会产生僵尸进程 |
| **如果此任务已经运行，以下规则适用** | `请勿启动新实例` | 防止重复执行 |

### 网络就绪触发方案

除了“计算机启动时”触发，还可以使用 **“发生事件时”** 作为触发器，监听网络连接就绪事件，确保在互联网连接建立后再启动热点：

- 日志：`Microsoft-Windows-NetworkProfile/Operational`
- 事件 ID：`10000`（网络连接已建立）
- 日志：`Microsoft-Windows-WLAN-AutoConfig/Operational`
- 事件 ID：`8001`（WLAN 已连接）

此方案可作为备用触发条件，与启动触发互补。


## 前置准备

### 确认关键系统服务

确保以下服务已启动且启动类型为“自动”：

| 服务名 | 显示名称 | 说明 |
|--------|----------|------|
| `SharedAccess` | Internet Connection Sharing (ICS) | 网络共享核心服务 |
| `WlanSvc` | WLAN AutoConfig | 无线网络自动配置服务 |

可通过 `services.msc` 检查并启动。

### 检查虚拟适配器

打开设备管理器 → 查看 → 显示隐藏的设备 → 网络适配器，找到 `Microsoft Wi-Fi Direct Virtual Adapter`，确保其已启用。若被禁用，右键选择“启用”。

### 关闭节能选项

在移动热点设置页面中，**关闭“节能”选项**。若开启此选项，无设备连接 5 分钟后热点会自动关闭。


## 常见问题与排查

### 热点无法启动

| 可能原因 | 排查方法 |
|----------|----------|
| 虚拟适配器被禁用 | 设备管理器 → 显示隐藏设备 → 启用 `Microsoft Wi-Fi Direct Virtual Adapter` |
| ICS 服务未运行 | `services.msc` 中确认 `Internet Connection Sharing` 为运行状态 |
| 无线网卡驱动不兼容 | 更新无线网卡驱动至最新版本 |
| 频段设置问题 | 网卡高级设置中将首选频带设为 `2.4GHz` |

### 任务计划执行失败

| 错误代码 | 含义 | 排查方法 |
|----------|------|----------|
| `0x1` | 脚本返回了退出代码 1 | 检查日志文件，确认脚本执行到哪一步失败 |
| `0x2` | 找不到指定的文件 | 确认操作选项卡中的 `.ps1` 路径完全正确 |
| `0x41301` | 任务已在运行 | 等待当前任务完成或手动终止 |

**0x1 错误的常见原因**：起始目录未设置。在操作选项卡的“起始于”中填写脚本所在目录路径。

### 热点启动后自动关闭

| 原因 | 解决方法 |
|------|----------|
| 节能选项已开启 | 移动热点设置中关闭“节能” |
| 电源管理关闭了网卡 | 设备管理器 → 网络适配器 → 属性 → 电源管理 → 取消“允许计算机关闭此设备以节约电源” |
| 电源模式过于激进 | 设置 → 系统 → 电源和电池 → 电源模式 → 最佳性能 |

### 日志查看

脚本执行日志位于：

```
C:\ProgramData\AutoHotspot\hotspot.log
```

包含每次执行的详细记录，包括连接配置文件名称、热点状态、执行结果等。

### 查看任务计划历史记录

双击任务 → “历史记录”选项卡，查看错误代码。若历史记录被禁用，可在任务计划程序中右键“任务计划程序库” → 启用所有任务历史记录。


## 验证方法

1. **手动测试**：在任务计划程序中右键任务 → “运行”，检查日志文件是否记录成功
2. **进程检查**：打开任务管理器 → “详细信息”页，查找 `powershell.exe` 进程
3. **重启验证**：重启计算机，停留在登录界面，用另一台设备搜索热点 SSID，确认能够连接
4. **日志确认**：重启后查看 `hotspot.log`，确认记录显示“热点启动成功”



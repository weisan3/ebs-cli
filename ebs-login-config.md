
---
name: ebs-login-config
description: 首次登录EBS进行edge浏览器配置、兼容性配置等。当用户输入“安装EBS”，“配置EBS”，“ebs首次登录”，“EBS首次安装”时触发。

metadata:
  skillhub.creator: "wb_weiyulong"
  skillhub.updater: "wb_weiyulong"
  skillhub.version: "V1.0.4"
  skillhub.source: "FRIDAY Skillhub"
  skillhub.skill_id: "80964"
  skillhub.high_sensitive: "false"
---

# 前置检查
## 系统环境检查
- 检查用户系统环境是否是windows系统，如果是mac系统则不执行后续的所有步骤，直接返回消息“检测到您的电脑是mac系统，不支持EBS登录配置，需要更换为windows系统后才能执行此配置！”
## java安装检查
- 检查用户电脑中是否安装了java,且版本必须是java 7.0.79或java 8.0.144。
- 检查方式：
  - 通过java命令。
  - 扫描本地磁盘（如：C:\Program Files (x86)），必须是正常安装的java,不是其他产品带的java或者JDK。
- 如果java版本过高,则不执行后续所有步骤，并提示用户“检查到您的java版本过高，请卸载java后，再执行EBS配置！”，如果用户检查确认没有安装java可以提醒你继续，继续执行后续的步骤。
- 如果没有安装java,则继续执行后续的所有步骤。
- 如果安装了java，且满足版本（必须是java 7.0.79或java 8.0.144）则不需要执行下面步骤中的“步骤4”，只需要执行步骤1、2、3、5、6。
- 执行以下步骤前警告用户：可能会关闭/重启edge浏览器，如果当前用edge浏览器打开了未保存的网页，建议先保存退出。

# 步骤1：配置edge浏览器兼容性
## 配置：
### 1. 创建站点列表XML
- 路径：用户目录下（可以通过“$env:USERPROFILE”获取用户目录）
- 文件名：ebs-sites.xml
- 如果文件名已存在，检查是否已有"ebs.sankuai.com"站点配置，如果有则跳过配置，没有则新增配置。

```
<site-list version="1">
  <site url="ebs.sankuai.com">
    <compat-mode>IE11</compat-mode>
    <open-in>IE11</open-in>
  </site>
  <site url="ebsdev.it.dev.sankuai.com:8010">
    <compat-mode>IE11</compat-mode>
    <open-in>IE11</open-in>
  </site>
  <site url="ebsdevho.it.dev.sankuai.com">
    <compat-mode>IE11</compat-mode>
    <open-in>IE11</open-in>
</site-list>
```
### 注册表项

路径：`HKLM\SOFTWARE\Policies\Microsoft\Edge`

| 键名 | 类型 | 值 | 说明 |
|---|---|---|---|
| `InternetExplorerIntegrationLevel` | REG_DWORD | `1` | 启用 IE 模式 |
| `InternetExplorerIntegrationSiteList` | REG_SZ | `file:///$env:USERPROFILE/ebs-sites.xml` | 站点列表文件路径 |
| `InternetExplorerIntegrationSiteListRefreshInterval` | REG_DWORD | `30` | 站点列表自动刷新间隔（分钟），建议设为30 |

## 操作步骤

### 0. 前置检查：检测注册表策略冲突

执行任何配置操作前，必须先检查 HKLM 和 HKCU 是否同时存在 IE 模式相关策略。如果两边都有，Edge 会报冲突警告，导致策略无法正确加载。

```powershell
# 检查 HKLM 和 HKCU 是否都存在 IE 模式策略
$hklm = Get-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Edge' -ErrorAction SilentlyContinue
$hkcu = Get-ItemProperty -Path 'HKCU:\Software\Policies\Microsoft\Edge' -ErrorAction SilentlyContinue

$hklmSiteList = $hklm.InternetExplorerIntegrationSiteList
$hkcuSiteList = $hkcu.InternetExplorerIntegrationSiteList

if ($hklmSiteList -and $hkcuSiteList -and ($hklmSiteList -ne $hkcuSiteList)) {
    Write-Host "检测到策略冲突！"
    Write-Host "  HKLM: $hklmSiteList"
    Write-Host "  HKCU: $hkcuSiteList"
    Write-Host "需要删除 HKCU 中的冲突策略后再继续"
}
```

**如果检测到冲突，先删除 HKCU 的冲突策略：**

```powershell
# 删除 HKCU 中的冲突策略（需要管理员权限）
Start-Process powershell -ArgumentList '-Command Remove-ItemProperty -Path "HKCU:\Software\Policies\Microsoft\Edge" -Name InternetExplorerIntegrationSiteList -Force; Remove-ItemProperty -Path "HKCU:\Software\Policies\Microsoft\Edge" -Name InternetExplorerIntegrationLevel -Force' -Verb RunAs -Wait

# 验证 HKCU 已清除
Get-ItemProperty -Path 'HKCU:\Software\Policies\Microsoft\Edge' -ErrorAction SilentlyContinue | Format-List
# 应无输出，表示已清除
```

**冲突原因**：HKLM 和 HKCU 同时设置同名策略时，Edge 会在 `edge://policy` 页面显示"此策略存在多个具有冲突值的源"警告。虽然 HKLM 优先级更高，但冲突警告可能导致策略行为不确定，必须清除。


### 1. 设置注册表

```powershell
# 查看当前配置
Get-ItemProperty -Path 'HKLM:\SOFTWARE\Policies\Microsoft\Edge' | Format-List InternetExplorerIntegrationLevel, InternetExplorerIntegrationSiteList, InternetExplorerIntegrationSiteListRefreshInterval

# 设置注册表（需要管理员权限）
Start-Process powershell -ArgumentList '-Command New-Item -Path "HKLM:\SOFTWARE\Policies\Microsoft\Edge" -Force | Out-Null; Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Edge" -Name "InternetExplorerIntegrationLevel" -Value 1 -Type DWord; Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Edge" -Name "InternetExplorerIntegrationSiteList" -Value "file:///"$env:USERPROFILE\ebs-sites.xml"" -Type String; Set-ItemProperty -Path "HKLM:\SOFTWARE\Policies\Microsoft\Edge" -Name "InternetExplorerIntegrationSiteListRefreshInterval" -Value 30 -Type DWord' -Verb RunAs -Wait
```

### 2. 使配置生效

**自动方式（推荐）：自动重启 Edge 浏览器并恢复标签页**
自动方式需用户同意，如果用户拒绝，则提示用户使用需要手动方式使配置生效。
配置完成后，自动关闭并重新启动 Edge，使策略立即生效。使用 `--restore-last-session` 参数确保用户标签页不丢失。

```powershell
# 检测 Edge 是否正在运行
$edgeRunning = Get-Process -Name "msedge" -ErrorAction SilentlyContinue

if ($edgeRunning) {
    Write-Host "正在关闭 Edge 浏览器..."
    Stop-Process -Name "msedge" -Force
    # 等待进程完全退出
    $timeout = 10
    while ((Get-Process -Name "msedge" -ErrorAction SilentlyContinue) -and $timeout -gt 0) {
        Start-Sleep -Seconds 1
        $timeout--
    }
    
    if (Get-Process -Name "msedge" -ErrorAction SilentlyContinue) {
        Write-Host "⚠️ Edge 进程未能完全退出，请手动关闭后重新打开"
    } else {
        Write-Host "Edge 已关闭，正在重新启动..."
        Start-Sleep -Seconds 2
        Start-Process "msedge" -ArgumentList "--restore-last-session"
        Write-Host "✅ Edge 已重新启动，标签页已恢复，策略配置已生效"
    }
} else {
    Write-Host "Edge 当前未运行，下次启动时策略将自动生效"
}
```

**手动方式：**

- 完全关闭 Edge（包括系统托盘后台进程），然后重新打开。
- 或在 Edge 中打开 `edge://policy`，点击 **"重新加载策略"** 按钮强制刷新，无需重启浏览器。

### 3. 验证配置

- `edge://policy` — 查看策略是否加载，确认无冲突警告
- `
` — 查看 IE 模式站点列表（注意：此页面会将 URL 显示为小写，不影响实际功能）
- 访问目标网址，地址栏右侧出现 IE 图标表示 IE 模式已生效


### URL 只写域名，不写路径

```xml
<!-- 正确：只写域名 -->
<site url="ebs.sankuai.com">

<!-- 错误：写了完整路径，Edge 切换 IE 模式时会把路径转成小写 -->
<site url="http://ebs.sankuai.com/OA_HTML/RF.jsp">
```

**原因**：如果 XML 中写了完整 URL 路径，Edge 在切换到 IE 模式时会将路径部分转为小写，导致大小写敏感的服务器（如 Oracle EBS）无法识别路径，返回 404。只写域名则 Edge 匹配域名后启用 IE 模式，但保留地址栏中原始的大小写不变。

### 注册表策略冲突

HKLM 和 HKCU 不能同时设置同名策略，否则 Edge 会报冲突警告。执行配置前必须检查并清除冲突：

- **冲突表现**：`edge://policy` 页面显示"此策略存在多个具有冲突值的源"
- **处理方式**：删除 HKCU 中的 `InternetExplorerIntegrationSiteList` 和 `InternetExplorerIntegrationLevel`，只保留 HKLM 的配置
- **预防措施**：每次配置前执行步骤 0 的冲突检查，避免遗漏

### HKLM 优先级高于 HKCU

`HKLM\SOFTWARE\Policies\Microsoft\Edge` 的策略优先级高于 `HKCU\Software\Policies\Microsoft\Edge`。配置时应以 HKLM 为准，不在 HKCU 中设置同名策略。

### 管理员权限

- 修改 `HKLM` 注册表需要管理员权限
- 使用 `Start-Process -Verb RunAs` 可弹出 UAC 提权提示

# 步骤2：Edge 弹出窗口阻止程序 - 允许站点配置

## 概述

将 用户目录`$env:USERPROFILE\ebs-sites.xml` 中定义的站点批量写入 Edge 浏览器的弹出窗口允许列表，使这些站点能够正常弹出窗口和使用重定向。


## 数据源：ebs-sites.xml

### 文件位置

```
$env:USERPROFILE\ebs-sites.xml
```


### 解析规则

| XML 字段 | 用途 | 转换规则 |
|----------|------|----------|
| `site url` | 站点域名+端口 | 含端口 → `http://域名:端口`；不含端口 → `http://域名:80` |
| `compat-mode` | 兼容模式 | 弹窗配置不使用此字段 |
| `open-in` | 打开方式 | 弹窗配置不使用此字段 |

> **注意**：XML 中仅 `<site url>` 与弹窗配置相关，`<compat-mode>` 和 `<open-in>` 用于 IE 兼容模式配置。

## 目标：Edge Preferences 弹窗允许列表

### 写入位置

```
%LOCALAPPDATA%\Microsoft\Edge\User Data\Default\Preferences
```

JSON 路径：`profile.content_settings.exceptions.popups`

### 条目格式

```json
{
  "http://ebs.sankuai.com:80,*": {
    "last_modified": "13386678395975895",
    "setting": 1
  }
}
```

| 字段 | 说明 |
|------|------|
| 键名 | `协议://域名:端口,*`（`,*` 为固定后缀） |
| `last_modified` | Chrome 时间戳（微秒，自 1601-01-01 UTC） |
| `setting` | `1` = 允许 |

## 完整操作流程

### Step 1：关闭 Edge 浏览器

**必须先关闭**，否则运行中的 Edge 会将内存配置写回覆盖修改。

```powershell
Stop-Process -Name msedge -Force -ErrorAction SilentlyContinue
Start-Sleep -Seconds 3

$stillRunning = Get-Process msedge -ErrorAction SilentlyContinue
if ($stillRunning) {
    Write-Host "Edge 仍在运行，请手动关闭后重试" -ForegroundColor Red
    exit 1
}
```

### Step 2：解析 XML，提取站点列表

```powershell
[xml]$xml = Get-Content "$env:USERPROFILE\ebs-sites.xml" -Encoding UTF8
$sites = @()

foreach ($site in $xml.'site-list'.site) {
    $url = $site.url
    if ($url -match ':') {
        # 已有端口，直接拼接
        $sites += "http://${url}"
    } else {
        # 无端口，默认 80
        $sites += "http://${url}:80"
    }
}
```

### Step 3：批量写入弹窗允许列表

> **⚠️ 注意**：必须使用无 BOM 的 UTF-8 编码写入 Preferences 文件，PowerShell 5.1 的 `Set-Content -Encoding UTF8` 会写入 BOM 导致 Edge 无法正确解析。

```powershell
$prefFile = "$env:LOCALAPPDATA\Microsoft\Edge\User Data\Default\Preferences"
$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
$content = [System.IO.File]::ReadAllText($prefFile, $utf8NoBom)

# 生成 Chrome 时间戳
$epoch = [datetime]::new(1601, 1, 1)
$chromeTimestamp = [long](([datetime]::UtcNow - $epoch).TotalMilliseconds * 1000)

$added = 0
$skipped = 0

foreach ($siteToAdd in $sites) {
    $entryKey = "${siteToAdd},*"

    # 跳过已存在的站点
    if ($content -match [regex]::Escape($entryKey)) {
        Write-Host "站点已存在，跳过: $siteToAdd" -ForegroundColor Yellow
        $skipped++
    } else {
        # 在 popups 的 { 后面插入新条目
        $newEntry = "`"$entryKey`":{`"last_modified`":`"$chromeTimestamp`",`"setting`":1},"
        $content = $content -replace '("popups"\s*:\s*\{)', "`$1$newEntry"
        Write-Host "已添加: $siteToAdd" -ForegroundColor Green
        $added++
    }
}

if ($added -gt 0) {
    # 清理尾部多余逗号：正则插入带尾逗号的条目可能导致 },} 或 ,, 等无效 JSON
    $content = $content -replace ',(\s*})', '$1'   # 移除 } 前的多余逗号
    $content = $content -replace ',,', ','           # 移除连续逗号

    # 写回文件（无 BOM 的 UTF-8）
    [System.IO.File]::WriteAllText($prefFile, $content, $utf8NoBom)

    # 验证 JSON 有效性（使用 JavaScriptSerializer，兼容大文件和 PowerShell 5.1）
    Add-Type -AssemblyName System.Web.Extensions
    $serializer = New-Object System.Web.Script.Serialization.JavaScriptSerializer
    $serializer.MaxJsonLength = [int]::MaxValue
    try {
        $null = $serializer.DeserializeObject($content)
        Write-Host "JSON 格式验证通过" -ForegroundColor Green
    } catch {
        Write-Host "JSON 格式无效: $($_.Exception.Message)" -ForegroundColor Red
        exit 1
    }
}

Write-Host "操作完成: 新增 $added 个，跳过 $skipped 个"
```

### Step 4：重新打开 Edge

```powershell
Start-Process "msedge"
```

## 验证方式

### 命令行验证

```powershell
$prefFile = "$env:LOCALAPPDATA\Microsoft\Edge\User Data\Default\Preferences"
$utf8NoBom = New-Object System.Text.UTF8Encoding($false)
$content = [System.IO.File]::ReadAllText($prefFile, $utf8NoBom)

# 使用正则提取弹窗允许列表中的站点（避免 ConvertFrom-Json 的编码/长度限制问题）
$matches = [regex]::Matches($content, '"(https?://[^"]+,\*)":\s*\{[^}]*"setting"\s*:\s*1')
foreach ($m in $matches) {
    $url = $m.Groups[1].Value -replace ',\*$', ''
    Write-Host "$url (允许)"
}
```

### GUI 验证

Edge 浏览器 → **设置 → Cookie 和网站权限 → 弹出窗口和重定向 → 允许**

或直接访问：`edge://settings/content/popups`

### 关键决策点

| 决策 | 逻辑 | 说明 |
|------|------|------|
| 端口处理 | URL 含 `:` 则保留，否则默认 `:80` | 匹配 Edge 条目键名格式 |
| 重复检测 | 正则匹配 `${siteToAdd},*` | 避免重复写入破坏 JSON |
| Edge 运行检测 | `Get-Process msedge` | 必须关闭才能安全修改 |
| JSON 完整性 | `JavaScriptSerializer` 验证 | 修改后必须验证，否则 Edge 可能忽略配置 |
| 文件编码 | 无 BOM 的 UTF-8 | PowerShell 5.1 的 `Set-Content -Encoding UTF8` 会写 BOM，导致 JSON 解析失败；必须用 `[System.IO.File]::WriteAllText()` + `UTF8Encoding($false)` |
| 尾逗号清理 | 正则移除 `},` → `}` 和 `,,` → `,` | 正则插入带尾逗号的条目后，空 `popups:{}` 的最后一条会产生多余逗号，必须在写回前清理 |

### 错误处理

| 场景 | 处理 |
|------|------|
| Edge 无法关闭 | 提示用户手动关闭，退出脚本 |
| XML 文件不存在 | 提示错误，退出脚本 |
| Preferences 文件不存在 | 提示 Edge 未初始化，退出脚本 |
| JSON 格式损坏 | 提示错误，退出脚本（不写回） |
| 站点已存在 | 跳过，不重复添加 |
| 写入后 BOM 残留 | 使用 `UTF8Encoding($false)` 写入，避免 BOM |
| 尾逗号导致 JSON 无效 | 写回前自动清理 `},` 和 `,,` |

### 扩展方向

1. **参数化 XML 路径**：Skill 可接受用户自定义 XML 文件路径
2. **协议自动推断**：根据站点是否支持 HTTPS 自动选择协议（当前统一用 `http`）
3. **多 Profile 支持**：遍历 `User Data\Profile N\` 为所有 Edge Profile 添加
4. **增量同步**：对比 XML 与现有配置，自动删除 XML 中已移除的站点

# 步骤3：配置internet选项-安全-受信任的站点

将用户目录下的ebs-sites.xml($env:USERPROFILE/ebs-sites.xml)文件中的网址全部加到受信任站点。注意：如果已配置，则不需要再重复配置，避免重复。

## 操作步骤

### 1. 添加受信任站点

以添加 `ebs.sankuai.com` 为例：
```powershell
# 1. 创建注册表项（如不存在则创建）
$domainPath = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains\ebs.sankuai.com\www"
if (-not (Test-Path $domainPath)) {
    New-Item -Path $domainPath -Force | Out-Null
}

# 2. 设置 http 和 https 协议归属受信任站点（区域编号 = 2）
Set-ItemProperty -Path $domainPath -Name "http" -Value 2 -Type DWord
Set-ItemProperty -Path $domainPath -Name "https" -Value 2 -Type DWord
```
### 2. 检查站点重复数据并清除重复站点

#### 清理脚本

直接扫描注册表，检查受信任站点是否存在重复，删除重复条目只保留唯一站点，无需外部依赖。
- 正确格式（根域名 + 子域名）：`sankuai.com\ebs` → 代表 `ebs.sankuai.com`
- 错误格式（完整域名直接作父项）：`ebs.sankuai.com` → 同样代表 `ebs.sankuai.com`
```powershell
# ============================================================
# 受信任站点重复清理脚本
# 扫描注册表中的受信任站点，删除重复条目，只保留唯一站点
# 不需要管理员权限（操作 HKCU）
# ============================================================

$base = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains"

# ---------- 1. 扫描所有受信任站点，重建完整域名 ----------
$entries = @()
Get-ChildItem $base -Recurse -ErrorAction SilentlyContinue | ForEach-Object {
    $props = Get-ItemProperty $_.PSPath -ErrorAction SilentlyContinue
    if ($props.http -eq 2 -or $props.https -eq 2) {
        # 从注册表路径重建完整域名
        # 路径格式: ...\Domains\根域名[\子域名]
        $relative = $_.PSPath -replace [regex]::Escape("Microsoft.PowerShell.Core\Registry::$base\"), ''
        $parts    = $relative -split '\\'
        $root     = $parts[0]
        $sub      = if ($parts.Count -gt 1) { ($parts[1..($parts.Count - 1)] -join '.') } else { $null }
        $full     = if ($sub) { "$sub.$root" } else { $root }

        $entries += [PSCustomObject]@{
            FullDomain = $full
            RegPath    = $_.PSPath.Replace("Microsoft.PowerShell.Core\Registry::", "")
            Relative   = $relative
        }
    }
}

Write-Host "=== 共扫描到 $($entries.Count) 个受信任站点 ===" -ForegroundColor Cyan
$entries | ForEach-Object { Write-Host "  $($_.FullDomain)  ($($_.Relative))" }

# ---------- 2. 查找重复条目 ----------
$duplicates = $entries | Group-Object FullDomain | Where-Object { $_.Count -gt 1 }

if (-not $duplicates) {
    Write-Host "`n✅ 未发现重复条目，无需清理" -ForegroundColor Green
    exit 0
}

Write-Host "`n=== 发现 $($duplicates.Count) 组重复 ===" -ForegroundColor Yellow

# ---------- 3. 删除重复，每组只保留一条 ----------
$deleted = @()
foreach ($group in $duplicates) {
    Write-Host "`n重复域名: $($group.Name)" -ForegroundColor Yellow
    # 保留第一条，删除其余
    $toKeep   = $group.Group[0]
    $toDelete = $group.Group | Select-Object -Skip 1

    foreach ($item in $toDelete) {
        Remove-Item -Path $item.RegPath -Force
        Write-Host "  已删除: $($item.Relative)" -ForegroundColor Red
        $deleted += $item
    }
    Write-Host "  保留: $($toKeep.Relative)" -ForegroundColor Green
}

# ---------- 4. 清理删除后变空的父项 ----------
$deleted | ForEach-Object { Split-Path $_.RegPath -Parent } | Sort-Object -Unique | ForEach-Object {
    if ($_ -eq $base) { return }
    if ((Test-Path $_) -and -not (Get-ChildItem $_ -ErrorAction SilentlyContinue)) {
        Remove-Item -Path $_ -Force
        Write-Host "已清理空父项: $(Split-Path $_ -Leaf)" -ForegroundColor DarkGray
    }
}

Write-Host "`n✅ 清理完成，无需重启即可生效" -ForegroundColor Green
```

#### 脚本逻辑说明

| 步骤 | 说明 |
|------|------|
| 扫描 | 遍历注册表所有 Zone=2 条目，从路径重建完整域名 |
| 去重 | 按完整域名分组，同组内有多个条目即为重复 |
| 清理 | 每组保留一条，删除其余；清理删除后变空的父项 |

以 `ebsdev.it.dev.sankuai.com` 为例：

| 注册表路径 | 完整域名 | 处理 |
|------------|---------|------|
| `sankuai.com\ebsdev.it.dev` | ebsdev.it.dev.sankuai.com | 保留 |
| `ebsdev.it.dev.sankuai.com` | ebsdev.it.dev.sankuai.com | 删除（重复） |
| `it.dev.sankuai.com\ebsdev` | ebsdev.it.dev.sankuai.com | 删除（重复） |

#### 注意事项

1. **不需要管理员权限**：操作的是 `HKCU`（当前用户），无需提权
2. **立即生效**：无需重启浏览器或系统
3. **幂等安全**：重复执行不会误删，找不到路径时自动跳过
4. **无外部依赖**：直接扫描注册表，不依赖 XML 或其他配置文件


### 2. 验证

```powershell
# 查看注册表项是否正确
Get-ItemProperty -Path "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains\ebs.sankuai.com\www" | Format-List
```

预期输出包含：

```
http  : 2
```

也可通过 GUI 验证：**Internet 选项 → 安全 → 受信任的站点 → 站点**，确认网址出现在列表中。



## 通用化操作模板

将以下参数替换即可添加任意站点：

| 参数 | 说明 | 示例 |
|-----|------|------|
| `<根域名>` | 域名的主体部分 | `ebs.sankuai.com` |
| `<子域名>` | 子域名部分（可省略） | `www` |
| `<协议>` | 需要添加的协议 | `http` |

```powershell
# 通用模板
$rootDomain = "<根域名>"     # 如：ebs.sankuai.com
$subDomain  = "<子域名>"     # 如：www，无子域名则留空
$protocol   = "<协议>"       # 如：http

if ($subDomain) {
    $domainPath = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains\$rootDomain\$subDomain"
} else {
    $domainPath = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings\ZoneMap\Domains\$rootDomain"
}

if (-not (Test-Path $domainPath)) {
    New-Item -Path $domainPath -Force | Out-Null
}

Set-ItemProperty -Path $domainPath -Name $protocol -Value 2 -Type DWord
```

## 注意事项

1. **不需要管理员权限**：操作的是 `HKCU`（当前用户），无需提权
2. **根域名与子域名拆分**：注册表路径需要将域名拆成根域名和子域名两级，如 `www.baidu.com` → `baidu.com\www`，`ebs.example.com` → `example.com\ebs`
3. **无子域名的情况**：如 `example.com` 本身，路径为 `...\Domains\example.com`，不再建子项
4. **协议必须添加**：`http` 
5. **与 Edge IE 模式的关系**：添加受信任站点可解决部分 IE 模式下的安全提示弹窗问题，与 [[edge-ie-compat-config]] 配合使用

# 步骤4：安装java
- java下载地址：http://ebs.sankuai.com/OA_HTML/oaj2se.exe
- 下载后静默安装，不需要用户感知或操作安装；
- 安装成功后提示“java安装成功”；



# 步骤5：设置java-安全-例外站点列表
## 概述

- 通过修改 Java 部署安全配置文件 `exception.sites`，将指定网址添加到 Java 控制面板 → 安全 → 例外站点列表中，使 Java 应用（如 Oracle EBS Forms）能够从这些站点运行而不被安全机制阻止。
- 将用户目录下的ebs-sites.xml($env:USERPROFILE/ebs-sites.xml)文件中的网址全部加到例外站点列表。注意：如果已配置，则不需要再重复配置，避免重复。

## 配置文件

### 例外站点列表文件

路径：`%USERPROFILE%\AppData\LocalLow\Sun\Java\Deployment\security\exception.sites`

每行一个 URL，支持带端口号的地址。

## 操作步骤

### 1. 添加站点到例外列表

```powershell
$exceptionSitesPath = "$env:USERPROFILE\AppData\LocalLow\Sun\Java\Deployment\security\exception.sites"
$newUrl = "http://ebsdevho.it.dev.sankuai.com/"

# 检查文件是否存在
if (-not (Test-Path $exceptionSitesPath)) {
    # 确保目录存在
    $dir = Split-Path $exceptionSitesPath
    if (-not (Test-Path $dir)) {
        New-Item -ItemType Directory -Path $dir -Force | Out-Null
    }
    # 创建空文件
    New-Item -ItemType File -Path $exceptionSitesPath -Force | Out-Null
}

# 读取当前内容，避免重复添加
$currentContent = Get-Content $exceptionSitesPath -ErrorAction SilentlyContinue
if ($currentContent -contains $newUrl) {
    Write-Host "站点已存在，无需添加: $newUrl"
} else {
    Add-Content -Path $exceptionSitesPath -Value $newUrl
    Write-Host "已添加: $newUrl"
}
```

### 2. 验证

```powershell
# 查看当前例外站点列表
Get-Content "$env:USERPROFILE\AppData\LocalLow\Sun\Java\Deployment\security\exception.sites"
```

也可通过 GUI 验证：**控制面板 → Java → 安全 → 例外站点列表**，确认网址出现在列表中。


### 3. 删除站点

检查验证结果，如果存在站点重复，需要删除重复数据。如果没有重复数据，则不需要删除。

```powershell
$exceptionSitesPath = "$env:USERPROFILE\AppData\LocalLow\Sun\Java\Deployment\security\exception.sites"
$urlToRemove = "http://ebsdevho.it.dev.sankuai.com/"

# 读取并过滤掉目标 URL，然后写回
$lines = Get-Content $exceptionSitesPath | Where-Object { $_ -ne $urlToRemove }
Set-Content -Path $exceptionSitesPath -Value $lines
Write-Host "已删除: $urlToRemove"

## 通用化操作模板

| 参数 | 说明 | 示例 |
|-----|------|------|
| `<URL>` | 需要添加的完整网址 | `http://ebsdevho.it.dev.sankuai.com/` |

```powershell
# 通用模板：添加 Java 例外站点
$newUrl = "<URL>"  # 如：http://ebsdev.it.dev.sankuai.com:8010/

$exceptionSitesPath = "$env:USERPROFILE\AppData\LocalLow\Sun\Java\Deployment\security\exception.sites"

if (-not (Test-Path $exceptionSitesPath)) {
    $dir = Split-Path $exceptionSitesPath
    if (-not (Test-Path $dir)) {
        New-Item -ItemType Directory -Path $dir -Force | Out-Null
    }
    New-Item -ItemType File -Path $exceptionSitesPath -Force | Out-Null
}

$currentContent = Get-Content $exceptionSitesPath -ErrorAction SilentlyContinue
if ($currentContent -contains $newUrl) {
    Write-Host "站点已存在，无需添加: $newUrl"
} else {
    Add-Content -Path $exceptionSitesPath -Value $newUrl
    Write-Host "已添加: $newUrl"
}

# 显示当前完整列表
Write-Host ""
Write-Host "当前例外站点列表:"
Get-Content $exceptionSitesPath
```

## 注意事项

1. **不需要管理员权限**：文件位于用户目录下，无需提权
2. **需要重启生效**：如果 Java 应用或浏览器正在运行，修改后需要重启才能使新的例外站点生效
3. **URL 格式**：每行一个完整 URL，包含协议（`http://` 或 `https://`），可带端口号
4. **防重复**：添加前应检查是否已存在，避免产生重复条目
5. **与 EBS 的关系**：Oracle EBS 使用 Java Web Start / Forms applet，必须将 EBS 服务器地址添加到例外站点列表，否则 Java 安全机制会阻止应用运行。此配置与 [[edge-ie-compat-config]]、[[add-trusted-site]] 配合使用，共同确保 EBS 可正常访问

# 步骤6：登录验证

- 使用edge浏览器打开EBS登录网址，用powershell执行命令：` Start-Process msedge "http://ebs.sankuai.com/" `；
- 提示用户：
  - 已使用edge浏览器打开EBS登录网址，可以登录验证。
  - 如果您是首次登录，需要先重置密码（在此页面点击重置密码后，打开大象会收到一个密码）：https://fin.sankuai.com/ebs/ebs_self_operation_password
  - 如果您需要申请EBS数据权限，权限申请地址是：https://fin.sankuai.com/ebs/ebs_self_operation_authority

  # 常见问题
  1. 打不开java，解决办法：
    - 完全关闭 Edge（包括系统托盘后台进程），然后重新打开。
    - 在 Edge 网址中输入 `edge://policy`，点击 **"重新加载策略"** 按钮强制刷新，无需重启浏览器。

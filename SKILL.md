---
name: windows-shell-safety
description: Windows 上执行 shell 命令、编写或运行 PowerShell 脚本时的防坑规则：&& 不被识别、中文输出乱码、curl/wget 是别名、退出码误判、重定向写出 UTF-16 等 AI 高频误操作。凡是在 Windows 环境执行命令、写 .ps1/批处理、或看到 PowerShell 报错（"'&&' is not a valid statement separator"、"无法将xxx识别为 cmdlet"、输出乱码）时都应使用本技能，即使用户没有明确提到 PowerShell。
---

# Windows Shell 安全规则（PowerShell 防坑）

Windows PowerShell 5.1（系统自带 powershell.exe）与 bash / PowerShell 7 的语义差异很大。按 bash 直觉写出的命令在 5.1 里要么直接报语法错，要么**静默执行但结果错误**（编码、别名、退出码）。错误经常表现为"看起来跑了，数据是坏的"，比报错更难发现——所以规则的核心是：先选对 shell，再按下面的清单避坑。

## 0. 先选对 shell

- 日常文件操作优先用环境提供的 POSIX shell（Git Bash）；只在需要 Windows 专属能力（注册表、服务、WMI、.ps1 脚本、计划任务）时才用 PowerShell。
- PowerShell 有两个世界，先分清：
  - `powershell.exe` = **5.1**，Windows 自带，坑最多；
  - `pwsh` = **7+**，支持 `&&`、`||`、utf8NoBOM，能用就用 7。
  - 判断：`$PSVersionTable.PSVersion`，或 `where.exe pwsh` 看是否装了 7。
- **5.1 没有 `&&` 和 `||`**。这是 AI 最常犯的错。`;` 只是顺序执行、不判断上一条成败。替代写法：

```powershell
# 错（5.1 直接语法报错）：cd C:\app && npm install
cd C:\app; if ($LASTEXITCODE -eq 0) { npm install }
# 或干脆拆成多次调用，每次检查退出码
```

## 1. 编码（中文系统第一大坑）

- 会话开始先设 UTF-8，否则 GBK 代码页下外部工具输出的中文全是乱码：

```powershell
[Console]::OutputEncoding = [System.Text.Encoding]::UTF8
```

- 读文件显式指定：`Get-Content -Encoding UTF8`。
- 5.1 中 `>` 重定向写出的是 **UTF-16LE**，很多工具读不了；`Out-File -Encoding utf8` 在 5.1 带 BOM。要干净的 UTF-8 用 `[System.IO.File]::WriteAllText($path, $text)`。
- **中文路径静默失败**：cv2.imread/imwrite、部分 ANSI 老工具对中文路径返回空/False 且不报错。对策：OpenCV 用 `imencode` + 二进制写，或换 ASCII 路径。写完务必检查返回值。

## 2. 别名陷阱（同名不同义）

- **`curl` / `wget` 在 5.1 里是 `Invoke-WebRequest` 的别名**，参数完全不同（`curl -s` 会报错）。要用真 curl 就写 `curl.exe`。
- `rm -rf x` 报错（没有 `-rf` 参数）→ `Remove-Item -Recurse -Force x`。
- `ls -la` → `Get-ChildItem -Force`；`export VAR=1` → `$env:VAR = '1'`（仅当前会话）。

## 3. 退出码与成败判断

- `$?` 只反映 cmdlet 的成败；**原生 exe 的退出码在 `$LASTEXITCODE`**。判断脚本/外部命令是否成功，以 `$LASTEXITCODE` 为准。
- git 等工具把进度信息写到 stderr，**stderr 有输出 ≠ 失败**，以退出码为准。
- 5.1 中 `2>&1` 配合 `$ErrorActionPreference='Stop'` 可能把 stderr 行误升级成异常；需要合并 stderr 时可用 `cmd /c "exe ... 2>&1"` 包一层。

## 4. 下载与网络

- `Invoke-WebRequest` 默认渲染进度条，大文件慢一个数量级。先设：

```powershell
$ProgressPreference = 'SilentlyContinue'
```

- 或直接用 `curl.exe -L -o <file> <url>`。

## 5. 引号与转义

- PowerShell 的转义符是**反引号 `` ` ``，不是反斜杠**；双引号会插值 `$var`，单引号是字面量——路径一律用单引号包。
- 复杂逻辑不要塞进 `-Command "..."`（多层引号转义必翻车）：写成临时 `.ps1`，再 `powershell -NoProfile -ExecutionPolicy Bypass -File script.ps1`。
- 从 bash 调用 `powershell -Command "..."` 时引号穿两层，最外层用单引号、内层变量改用 `\"` 转义，或干脆走临时脚本。

## 6. 让输出对 AI 可读

- PowerShell 默认表格格式会**截断长列**（显示成 `...`）。要看全内容：

```powershell
Get-Process | Select-Object Name, Id, Path | ConvertTo-Json -Compress
# 或
Get-ChildItem -Recurse | Out-String -Width 4096
```

## 7. bash 习惯 → PowerShell 对照

| bash 写法 | PowerShell 等价 |
|---|---|
| `a && b` (5.1) | `a; if ($LASTEXITCODE -eq 0) { b }` |
| `grep pat f` | `Select-String -Pattern pat f` |
| `find . -name "*.log"` | `Get-ChildItem -Recurse -Filter *.log` |
| `which x` | `Get-Command x` |
| `cat f` | `Get-Content f` |
| `sleep 5` | `Start-Sleep -Seconds 5` |
| `kill -9 <pid>` | `Stop-Process -Id <pid> -Force` |
| `tail -f log` | `Get-Content log -Wait -Tail 20` |
| `export VAR=1` | `$env:VAR = '1'`（会话级） |
| `ln -s t l` | `New-Item -ItemType SymbolicLink -Path l -Target t`（需管理员/开发者模式） |

## 8. 路径与环境变量

- 含空格路径必须加引号；超过 260 字符的路径加 `\\?\` 前缀。
- 环境变量读法是 `$env:JAVA_HOME`，不是 `$JAVA_HOME`；持久化设置用：

```powershell
[Environment]::SetEnvironmentVariable('JAVA_HOME', 'C:\jdk', 'User')
```

- 运行 .ps1 被策略拦截时：`powershell -ExecutionPolicy Bypass -File x.ps1`，不要去全局改执行策略。

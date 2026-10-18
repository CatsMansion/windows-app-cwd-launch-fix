---
name: windows-app-cwd-launch-fix
display_name: Windows 程序启动目录崩溃 启动器加固 CWD 相对路径 PyInstaller 双击exe崩溃
display_name_en: Windows App CWD Launch Fix
description: 排查并修复「Windows 下打包好的程序，双击就崩，报找不到 res/xxx 之类的相对路径文件」这一类问题。含四步排除法（行尾 / start 继承 / 文件是否真的缺失 / CWD 归因）、A-B 对照复现、以及一个带自检分支的加固启动器模板。
description_zh: Windows 程序启动目录崩溃排查与修复（启动崩溃/双击exe崩溃/CWD/工作目录/相对路径/PyInstaller/FileNotFoundError/res路径找不到/启动器/launch.cmd/bat/cwd不对/换个目录就崩/打包给客户跑不起来）：四步排除法 + A-B 对照复现 + 自检型启动器模板。
description_en: Diagnose and fix Windows packaged apps that crash on launch because relative resource paths resolve against the process working directory. Includes an elimination checklist, an A/B reproduction method, and a self-checking launcher template.
category: 开发工具
version: 1.0.0
author: WorkBuddy
agent_created: true
---

# Windows 程序启动目录（CWD）崩溃：排查与加固

## 什么时候用

打包成 EXE 的 Windows 程序（PyInstaller / Nuitka 等），出现下面任一现象：

- 用户双击就弹错误框，报 `FileNotFoundError: ...` 找不到某个资源文件
- 报错路径看着像相对路径（`res/xxx`、`assets/xxx`、`resources/xxx`），没有盘符
- 你自己在终端跑没事，用户双击就崩（或者反过来）
- 换一台电脑 / 换个文件夹 / 用快捷方式启动就崩
- 报错经过 `settings.py` / `config.py` 这类模块，并且在启动早期就炸

**判据**：那个"找不到"的文件**其实存在于程序目录里**。如果文件真不存在，那是打包漏文件，不是这个问题。

## 根因

打包程序的资源路径有两种写法：相对路径（跟**进程工作目录 CWD** 走）和基于程序自身位置的绝对路径。
前者写起来省事，于是很多项目在 Windows 分支直接 `basedir = ''`，让所有资源路径保持相对。

结果是：**程序能不能找到自己的资源，取决于「从哪个目录把它拉起来的」，而不是「它被放在哪」。**
CWD 不对 → 启动瞬间必崩。

**关键事实：双击 exe 本身也不保证 CWD 正确。** 不要假设"用户双击了就等于 CWD 对"。
快捷方式、开始菜单搜索、Win+R、把 exe 拖进终端、在别的目录里命令行启动，都可能给出错误的 CWD。

## 四步排除法

**不要跳过前三步直接下结论**，否则很容易误判成别的问题、改错东西。

### 第 1 步：确认文件真的存在

去程序目录里查那个报错文件。

- **存在** → 继续，这是 CWD 问题
- **不存在** → 是打包漏文件，去查打包配置里资源有没有被收进去。与本 skill 无关

### 第 2 步：排除批处理行尾问题（如果你已经用了 .cmd 启动器）

LF-only 的 `.cmd` 在某些写法上（标签、`for`、括号块）会被 cmd.exe 解析坏。
**实测结论：纯 `cd /d "%~dp0"` 和单行 `for ... do ...` 在 LF 下也能正常解析**，
所以行尾通常不是元凶——但规范成 **CRLF + 纯 ASCII** 永远是对的，成本为零。

自查：`cat -A file.cmd`，正常应看到 `^M$`；只有 `$` 说明是 LF。

### 第 3 步：排除 `start` 吞掉工作目录

如果启动器里用 `start` 拉起程序，验证子进程的 CWD 有没有正确继承。

用一个无害的替身程序（`cmd.exe` 自己）代替真程序，让它把 CWD 写进文件：

```bat
@echo off
cd /d "%~dp0"
start "" /wait cmd.exe /c "cd > out.txt"
```

然后**从一个错误的工作目录**调用这个 .cmd，看 `out.txt` 落在哪。
落点正确 = `start` 没问题，问题在别处。

### 第 4 步：A/B 对照复现（决定性）

同一个 exe，**只改 CWD**，跑两次：

| 用例 | 工作目录 | 预期 |
|---|---|---|
| A | 程序所在目录 | 正常启动 |
| B | 别的目录（比如 `C:\Users\<用户名>`） | 复现用户报错 |

B 用例能复现出**与用户截图逐字一致**的 traceback → 归因成立，可以动手修了。
复现不出来 → 你的假设是错的，回去看第 1 步。

用 Python 跑最省事（能拿到 stdout 且有超时）：

```python
import subprocess, time
proc = subprocess.Popen([EXE], cwd=WRONG_DIR,
                        stdout=subprocess.PIPE, stderr=subprocess.STDOUT,
                        creationflags=0x08000000)  # CREATE_NO_WINDOW
time.sleep(15)
alive = proc.poll() is None
out, _ = proc.communicate() if not alive else (proc.stdout.read(), None)
# 判据：CWD 正确时进程能活下来；错误时很快退出并吐出 traceback
```

## 加固的启动器模板

放在程序目录里（和 exe 同级），命名 `launch.cmd`。**纯 ASCII + CRLF。**

```bat
@echo off
rem  >>>  DOUBLE-CLICK THIS FILE TO START  <<<
rem  The app resolves resources against the PROCESS WORKING DIRECTORY.
rem  Starting it from anywhere else crashes with a missing-file error.
rem  This launcher forces the working directory to its own folder.
rem  ASCII-only, CRLF line endings.

setlocal
cd /d "%~dp0"

if not exist "%~dp0res\language\language.json" goto missing
if not exist "%~dp0_internal" goto missinginternal

rem  /d sets the child's startup directory too -- belt and suspenders.
for %%F in ("%~dp0*.exe") do start "" /d "%~dp0" "%%~fF"
goto done

:missing
echo.
echo  [ERROR] A required resource folder was not found next to this launcher.
echo          The package is incomplete or was not fully unzipped.
echo.
echo  Folder: %~dp0
echo.
pause
exit /b 1

:missinginternal
echo.
echo  [ERROR] The "_internal" folder is missing next to this launcher.
echo          Unzip the package completely. Do not run the .exe from inside
echo          the .zip preview.
echo.
echo  Folder: %~dp0
echo.
pause
exit /b 1

:done
endlocal
```

设计要点：

- `cd /d "%~dp0"` ＋ `start "" /d "%~dp0"` **双保险**，任一失效都不会错
- **启动前自检**关键的资源文件与运行时目录，缺失就打印人话错误并 `pause`
  —— 客户看到的是"包没解压完整"，不是一屏 Python traceback
- 用 `for %%F in ("%~dp0*.exe")` 而不是硬编码 exe 名
  —— 这样**启动器里不含非 ASCII 字符**，能扛住中文 exe 名
- 报错文案里**不要出现括号**（`(` `)`），会被 `( )` 块解析吃掉

### 必须实测的三个分支

改完启动器，三个分支都要跑一遍，缺一个都可能在客户手上炸：

1. 缺关键资源文件 → 应打印错误并 `pause`，退出码 1
2. 缺运行时目录（`_internal` 等） → 同上
3. 一切齐全 → 应成功拉起程序

分支 1、2 的验法：把启动器拷到一个空目录，再补上部分文件，逐档试。

### 端到端确认（很有用）

从一个**错误的工作目录**调用启动器，然后在**同一条命令内**等它跑完再检查痕迹：

- 存档 / 日志文件的时间戳是否刚被刷新
- 运行时目录是否出现新的临时文件

> ⚠️ 不要用「进程还在不在」做长期验证——很多执行环境会回收分离的子进程。
> 在命令结束那一刻检查落盘痕迹最可靠。

## 配套的收尾动作

修好启动器只是第一步，还有三件事：

1. **告诉用户"点这个，不要点 exe"**，并把这句话写进面向用户的说明里。
   用户不看文件名，只看图标；所以最好把 exe 的入口藏起来，或让启动器成为唯一一眼可见的东西。
2. **客户说明里单独开一节讲启动方式**，把会崩的几种姿势（快捷方式 / 搜索框 / Win+R / 拖进终端 / 别的目录）
   全部点名拉黑，并附上报错原文，让用户能自己对号入座。
3. **提醒存档位置**。这类程序的存档通常也在 CWD 下，**移动程序目录 = 丢存档**，
   说明里要写清备份方法。

## 常见误判

| 误判 | 实际 |
|---|---|
| "文件缺失，打包漏了" | 先确认文件是否存在，多半存在 → 是 CWD |
| "批处理行尾不对" | LF 下简单语句也能跑；行尾是次要嫌疑，别在这儿耗太久 |
| "`start` 吞了工作目录" | 实测 `start` 会正确继承；用替身程序验证，别猜 |
| "用户双击了所以 CWD 一定对" | **错。双击 exe 不保证 CWD 正确。** 这是最常见的误判 |
| "用快捷方式最稳" | 快捷方式能不依赖批处理，但**创建 `.lnk` 需要 COM 实例化，可能被安全策略拦截**；被拦就用 `.cmd` 转发 |

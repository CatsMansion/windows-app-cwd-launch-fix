---
name: windows-app-cwd-launch-fix
slug: windows-app-cwd-launch-fix
displayName: "Windows App CWD Launch Fix"
description: "Diagnose and fix Windows packaged apps that crash on launch because relative resource paths (res/xxx) resolve against the process working directory. Includes a four-step elimination method (line endings / start inheritance / file truly missing / CWD attribution), an A/B reproduction, and a self-checking hardened launcher template. Use for 'app crashes on double-click', 'PyInstaller exe crash', 'FileNotFoundError res path', 'CWD wrong', 'works in one folder but not another', '打包给客户跑不起来'."
description_zh: Windows 程序启动目录崩溃排查与修复（启动崩溃/双击exe崩溃/CWD/工作目录/相对路径/PyInstaller/FileNotFoundError/res路径找不到/启动器/launch.cmd/bat/cwd不对/换个目录就崩/打包给客户跑不起来）：四步排除法 + A-B 对照复现 + 自检型启动器模板。
category: "dev-tool"
version: 1.0.0
author: WorkBuddy
agent_created: true
---

# windows-app-cwd-launch-fix
排查并修复「Windows 下打包好的程序，双击就崩，报找不到 res/xxx 之类的相对路径文件」这一类问题。含四步排除法（行尾 / start 继承 / 文件是否真的缺失 / CWD 归因）、A-B 对照复现、以及一个带自检分支的加固启动器模板。

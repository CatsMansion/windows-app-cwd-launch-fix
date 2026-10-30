# windows-app-cwd-launch-fix

An Agent Skill for the classic Windows packaging failure: **the app crashes on double-click with `FileNotFoundError: res/...`, but the file is clearly there.**

## Root cause

Plenty of packaged apps resolve resources relative to the **process working directory**. In the DyberPet case, `settings.py` sets `basedir = ''` on `win32`, so every `res/...` path is relative — and the CWD decides whether the app starts at all.

Double-clicking usually sets the CWD to the app folder, which is why it always works on the developer machine. It breaks when a shortcut carries a different "Start in" value, when another process `start`s the exe (children inherit the CWD), or when Task Scheduler launches it from `System32`.

## Four-step elimination, in order

1. **Is the file actually missing?** Reconcile counts instead of eyeballing — zip entry count versus extracted file count versus expected extras. Real case: 1785 entries produced 1803 files, which is 1785 plus 18 mod files. Nothing was missing.
2. **Line endings and encoding.** `.cmd` and `.bat` must be CRLF and pure ASCII. LF breaks `goto`; a UTF-8 BOM turns line 1 into an unrecognised command.
3. **`start` inheritance.** `start "" app.exe` inherits the CWD. `start "" /d "%~dp0" app.exe` does not.
4. **CWD attribution — the only real evidence.** Run A and B for 8 seconds each. From an unrelated directory it must reproduce the customer traceback exactly; from the app folder it must start.

## Hardened launcher

`scripts/launch.cmd` does four things: `cd /d "%~dp0"` to pin the CWD, pre-flight existence checks with human-readable `:missing` branches, `start "" /d "%~dp0"` as a second lock on the child process, and ASCII plus CRLF throughout.

The `:missinginternal` branch specifically catches "double-clicked inside the zip preview", which otherwise drops the customer into a traceback.

## Acceptance

Start it from four places: Explorer double-click, a desktop shortcut, `Win + R`, and a cmd window in a different directory. Then reverse-test: rename `res` and confirm the launcher reports the missing-file error instead of dying silently.

## Scope

This is a tourniquet, not a cure. The durable fix is `basedir = os.path.dirname(sys.executable)`. But when the package is already in a customer hands, the tourniquet ships first.

MIT.

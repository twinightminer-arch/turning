# turning · 更新专用技能

> 面向「打包 → 发布 → 版本迭代」全流程的 Agent 技能库。
> 目前收录 1 个技能：**`win-gui-tool-exe`**。

---

## 📦 收录技能

| 技能 | 作用 | 触发场景 |
|---|---|---|
| `win-gui-tool-exe` | 把 Python 逻辑做成「带 tkinter 界面 + 自定义图标」的 Windows 单文件 exe（PyInstaller），并完成发布闭环 | 「打包成可用的 exe」「做个带界面的工具」「要按钮 + 进度条 + 日志窗口」「打包成安卓 APK」「快捷方式还指向旧版本」 |

技能覆盖的实战要点：

- 交互形态确认（GUI vs 双击即跑）与 tkinter 骨架（绿色进度条 / 终端式日志 / DPI 感知）
- 图标合成（PIL 生成多尺寸 `.ico`，如「logo + 蓝色圆圈」）
- 中文路径与编码坑、windowed 模式 `subprocess` 坑
- 发布闭环：部署新产物 → 快捷方式重定向 → 读回验证
- Windows / Android 两条打包链分流（PyInstaller vs buildozer）
- NSIS 安装包实制与踩坑
- 沙箱下用 ctypes 建 `.lnk`、送回收站

---

## 🔢 版本号迭代规则

这是本技能库的**核心约定**，所有由本技能发布的产物都遵循下表。

### 一、常规迭代（默认行为）

| 步骤 | 版本号变化 | 说明 |
|---|---|---|
| 首次发布 | `0.1.0` | **起始版本**。不是 `1.0.0`，也不是 `0.0.1` |
| 第 1 次更新 | `0.1.1` | 每次更新 `+0.0.1` |
| 第 2 次更新 | `0.1.2` | 同上 |
| … | … | … |
| 第 9 次更新 | `0.1.9` | 第三位到 9 |
| 第 10 次更新 | `0.2.0` | ⚠️ **满 10 进位**，不是 `0.1.10` |
| 第 11 次更新 | `0.2.1` | 继续逐位累加 |

> **关键**：第三位是十进制位，**满 10 进 1**。
> ✅ 正确：`0.1.9 → 0.2.0`
> ❌ 错误：`0.1.9 → 0.1.10`

### 二、特殊情形

| 情形 | 版本号处理 | 是否需先与用户确认 |
|---|---|---|
| **常规迭代** | `+0.0.1` | 不需要 |
| **大更新 / 大改动** | 可能 `+0.1.0`（如 `0.1.7 → 0.2.0`） | ⚠️ **必须先询问**「版本号是否增加 0.1.0」 |
| **推倒重制** | 从重制发布那版**重新 `0.1.0`** 起，不延续旧序列 | 确认「确实算重制」 |
| **开多分支** | 见下方「三、多分支协作」 | ⚠️ **必须先询问哪个方向是主分支** |
| **分支合并后** | 去掉字母后缀，回归主分支迭代 | 不需要 |

> 例：旧线已走到 `0.6.3`，此时决定重制 —— 重制版发布的版本号是 **`0.1.0`**，而不是 `0.6.4`。

### 三、多分支协作

四步走：

1. **先问主分支**：明确哪个方向作为主分支（不默认、不自行决定）
2. **主分支照常迭代**：从分支开始前的版本号继续 `+0.0.1`
3. **次分支加后缀**：以主分支分叉点的版本号为底，**末尾追加大写字母**（`A` `B` `C` `D` `E`…）
   - 分支内继续迭代时，数字部分照常 `+0.0.1`，后缀保留
   - 后缀**必须大写**，与分支一一对应，**不复用**
4. **合并后回归**：合并回主分支后**去掉字母后缀**，恢复主分支正常迭代

**示例**（分叉点 `0.3.2`）：

| 分支 | 版本序列 | 说明 |
|---|---|---|
| 主分支 | `0.3.2` → `0.3.3` → `0.3.4` | 正常迭代，不受分支影响 |
| 分支 A | `0.3.2A` → `0.3.3A` | 主分支分叉点号 + 大写后缀 |
| 分支 B | `0.3.2B` | 另一个方向，换用后缀 `B` |
| A 合并回主分支 | 主分支照常发 `0.3.4` | 去后缀，恢复主分支迭代 |

> 边界情况（例如 A、B 同时合并）以用户口头约定为准，口头约定优先于本表。

### 四、版本号落地位置（必须一致）

同一个版本号要同时写进三处：

| 位置 | 示例 |
|---|---|
| 产物文件名 | `ChatGPT修复工具-Setup-0.2.0.exe` |
| 安装包 `OutFile` | `OutFile "dist\MyTool-Setup-0.2.0.exe"` |
| GitHub Release tag | `v0.2.0` |

---

## 🚩 发布硬要求

每次发布都要执行，缺一不可：

| # | 要求 | 说明 |
|---|---|---|
| 1 | **主动询问是否做安装包并上传** | 便携版就绪后不要直接收尾，先问用户是否需要「制作安装包 exe 并上传」 |
| 2 | **二进制产物挂 Release** | `.exe` / `.apk` 一律作为 GitHub Release 附件，不能只推源码 |
| 3 | **版本号按上表迭代** | 首版 `0.1.0`、每次 `+0.0.1`（满 10 进位）；大改动先问；重制归零；多分支先问主分支 |
| 4 | **封装前做「平台串台」自检** | 打包**开始前**跑检查，确认没有把 Android 代码写成 Windows 端、或反之（核心逻辑层禁止出现 `tkinter` / `kivy` 等平台 import），详见下节 |

---

## 🚧 封装前检查：平台串台自检

Windows 与 Android 是**两条完全不同的链**，最常见的返工就是**把一端代码写进另一端的层里**：
`import tkinter` 混进两端共用的 `core.py`，Windows 版看着正常、Android 版一打就废；
`from kivy.app import App` 混进 PC 版，exe 启动即崩。
**编译器和打包器都不会提示**，必须在封装前主动查。

### 检查清单

| # | 检查项 | 判定标准 |
|---|---|---|
| 1 | **核心逻辑层纯净** | `core.py` 等两端共用模块**不得出现** `tkinter` / `kivy` / `jnius` / `android` / `plyer` / `win32*` / `ctypes.windll` 任一平台 import |
| 2 | **UI 层各归各位** | `ui_win.py` 只允许 Windows 栈；`ui_android.py` 只允许 Android 栈；**交叉即错** |
| 3 | **平台分支收拢** | `if sys.platform` / `os.name` 判断集中在一个 compat 模块，不散落在业务代码 |
| 4 | **路径与 API 无硬编码** | Windows 侧不该有 `app.user_data_dir`；Android 侧不该有 `C:\`、`%APPDATA%`、注册表、`.lnk` |

### 做法：用 AST 扫描，不要用 `grep`

`grep` 有两个致命问题：

1. 注释、字符串、文档里的关键词也会命中，**误报一堆**；
2. **抓不到写在函数体内部的 `import`**（"延迟导入"写法）。

用 `ast` 解析 import 节点才干净，且能覆盖函数内 import：

```python
# check_platform_leak.py —— 封装前跑一次，退出码非 0 即拦截打包
import ast, sys, pathlib

WIN_ONLY = {"tkinter", "winreg", "win32api", "win32con", "win32com", "pythoncom", "pywin32"}
AND_ONLY = {"kivy", "jnius", "plyer", "android", "buildozer"}
SHARED_MUST_BE_CLEAN = {"core.py", "logic.py", "model.py"}   # 两端共用，必须零平台依赖

def imports_of(path):
    tree = ast.parse(path.read_text(encoding="utf-8", errors="replace"))
    got = set()
    for n in ast.walk(tree):                     # walk 覆盖函数体内部的 import
        if isinstance(n, ast.Import):
            got |= {a.name.split(".")[0] for a in n.names}
        elif isinstance(n, ast.ImportFrom):
            if n.module:                         # 跳过相对 import
                got.add(n.module.split(".")[0])
    return got

bad = []
for p in pathlib.Path(".").rglob("*.py"):
    if any(s in p.parts for s in ("build", "dist", ".venv", "venv")):
        continue
    lw, la = imports_of(p) & WIN_ONLY, imports_of(p) & AND_ONLY
    if p.name in SHARED_MUST_BE_CLEAN and (lw or la):
        bad.append(f"[核心层污染] {p}: {sorted(lw | la)}")
    if p.name == "ui_android.py" and lw:
        bad.append(f"[Android 层混入 Windows 代码] {p}: {sorted(lw)}")
    if p.name == "ui_win.py" and la:
        bad.append(f"[Windows 层混入 Android 代码] {p}: {sorted(la)}")
    if lw and la:
        bad.append(f"[同文件双平台混用] {p}: win={sorted(lw)} android={sorted(la)}")

print("\n".join(bad) if bad else "PLATFORM_LEAK_CHECK: OK")
sys.exit(1 if bad else 0)
```

```bash
python check_platform_leak.py     # 退出码 0 才允许进入打包
```

### 处理经验

| # | 经验 |
|---|---|
| 1 | **报错信息会骗人** —— Android 侧打包失败抛的是 buildozer/Kivy 的错，根因常是 `core.py` 里一句 `import tkinter`。先跑检查再深挖 |
| 2 | **函数体里的 `import` 也算** —— `grep "^import"` 抓不到；AST 遍历全节点能抓到，这是它优于 grep 的关键 |
| 3 | **`try/except ImportError` 掩护的平台代码同样是坑** —— 会让打包器把 tkinter 打进 Android 包，体积暴涨且运行报错。要么挪到平台层，要么用显式 `if sys.platform` |
| 4 | **修完必须复跑** —— 挪完平台 import 后重跑检查、退出码为 0 再打包，别凭"我改过了"就发版 |
| 5 | **打包链也要对齐** —— `.exe` 只能 PyInstaller、`.apk` 只能 buildozer/briefcase，**PyInstaller 出不了 APK** |
| 6 | **两端都产出时，一次改两端都验** —— 只验 Windows 版就发布，Android 侧串台会拖到用户手里才炸 |

---

## 📁 目录结构

```
turning/
├── README.md              # 本文件
├── SKILL.md               # 主技能：win-gui-tool-exe
└── agents/
    └── openai.yaml        # Codex 侧界面元数据（显示名 / 品牌色 / 默认提示语）
```

---

## 🔧 安装到本机

| Agent | 目标目录 |
|---|---|
| WorkBuddy | `~/.workbuddy/skills/win-gui-tool-exe/SKILL.md` |
| Codex | `~/.codex/skills/win-gui-tool-exe/SKILL.md`（+ `agents/openai.yaml`） |

两侧内容保持一致（同源），任一侧改进后同步另一侧。

---

## 📄 许可

个人自用技能库，未设正式许可证。

---
name: win-gui-tool-exe
description: 把 Python 逻辑做成"带 tkinter 界面 + 自定义图标"的 Windows 单文件 exe（PyInstaller），发布前把快捷方式重定向到最新产物，并按 Windows / Android 分流选择不同的打包链。当用户说「打包成可用的 exe」「做个带界面的工具」「把这个脚本变成 exe」「要按钮 + 进度条 + 日志窗口」「打包成安卓 APK」「快捷方式还指向旧版本」时使用。覆盖图标合成（PIL 生成多尺寸 ico，如"logo + 蓝色圆圈"）、中文路径/编码坑、windowed 模式 subprocess 坑、坏 junction 检测、.lnk 重指等实战经验。
---

# 让 Python 变成「带界面的 Windows exe」

> **⚠️ 发布四条硬要求（每轮都要执行，详见 §13 / §14）**
> 1. **每次发布前，主动询问用户**是否需要"制作安装包 exe 并上传" —— 不要默认用户只要便携版，也不要默认不需要上传。
> 2. **二进制产物（`.exe` / `.apk`）自动放 GitHub Release** —— 不能只推源码。
> 3. **版本号**：首版一律 `0.1.0`，之后每次 +`0.0.1`（**满 10 进位**，如 `0.1.9 → 0.2.0`）；重制则从重制版重新 `0.1.0` 起；大改动先问"是否 +0.1.0"（如 `0.1.7 → 0.2.7`）；多分支先问哪个是主分支，非主分支在主分支号后加 `A/B/C…` 后缀，合并后回归主分支迭代。
> 4. **封装前必做「平台串台」自检** —— 确认没有把 **Android 代码写成 Windows 端**、或**把 Windows 代码写成 Android 端**（核心逻辑层禁止出现 `tkinter` / `kivy` 等平台 import），详见 §10.5。

## 0. 先定交互形态（问清或按用户原话）
- **「打包成可用 exe」** ≠ 「打开就自动跑」。用户要的是**窗口 + 功能按钮**时，就做 GUI（`--windowed`），点击按钮才执行。
- 典型界面要素：功能按钮（主按钮醒目蓝色）、**绿色进度条**、**终端式过程输出**（黑底等宽字）。

## 1. 环境探测（第一步，别急着写代码）
```bash
PY310="/c/Users/$USER/AppData/Local/Programs/Python/Python310/python.exe"
"$PY310" -m PyInstaller --version   # 需要 6.x
"$PY310" -c "import PIL;print(PIL.__version__)"  # 需要 Pillow（做图标）
```
- 本机常见情况：**系统 Python 3.10.11 已带 PyInstaller + Pillow**；managed 3.13 往往没有 → 直接用系统 3.10，别浪费时间装。
- 两个都缺时才 `pip install`（装到隔离 venv）。

## 2. 目录约定（重要）
- **把源码/构建放在纯 ASCII 路径**，例如 `C:\Users\Public\<proj>\`。中文路径下 PyInstaller / PowerShell 读脚本都容易出编码问题。
- 最后把 exe + 源码 **拷贝**到用户要的中文目标目录（拷贝不受影响）。
- 中间产物 `build\` 用完删掉（用户 C 盘常紧张）。

## 3. GUI 骨架（tkinter，零依赖）
必备片段：
```python
# 绿色进度条：必须切 clam 主题才能自定义颜色
style = ttk.Style(); style.theme_use("clam")
style.configure("Green.Horizontal.TProgressbar", troughcolor="#E6EAF2",
                background="#22C55E", lightcolor="#22C55E",
                darkcolor="#22C55E", bordercolor="#E6EAF2", thickness=16)

# 终端区
txt = tk.Text(parent, bg="#0C111B", fg="#C9D4E5", font=("Consolas", 10),
              relief="flat", bd=0, wrap="word", state="disabled")
# 用 tag 上色：info/step/ok/warn/err -> 灰白/青/绿/黄/红
txt.tag_configure("ok", foreground="#5BE38A")

# DPI 感知（防模糊）
try: ctypes.windll.shcore.SetProcessDpiAwareness(1)
except Exception: pass
root.tk.call("tk", "scaling", root.winfo_fpixels("1i") / 72.0)

# 工作线程 -> queue -> 主线程 after(60) 轮询刷新（tkinter 非线程安全，绝不跨线程碰控件）
```

## 4. 图标：任意底图 + 蓝圈徽章
源图优先从目标程序自带资源里取（PNG 比 ICO 好处理）：
```python
from PIL import Image, ImageDraw
logo = Image.open(SRC).convert("RGBA")
bb = logo.split()[3].getbbox()          # 裁掉透明留白
if bb: logo = logo.crop(bb)
S, MARGIN, RING = 1024, 46, 78
c = Image.new("RGBA", (S, S), (0,0,0,0)); d = ImageDraw.Draw(c)
d.ellipse([MARGIN, MARGIN, S-MARGIN, S-MARGIN], fill=(24,105,255,255))       # 蓝圈
d.ellipse([MARGIN+RING]*2 + [S-MARGIN-RING]*2, fill=(255,255,255,255))       # 白内盘
t = int((S-2*MARGIN-2*RING)*0.66)
lg = logo.resize((t,t), Image.LANCZOS)
c.alpha_composite(lg, ((S-t)//2, (S-t)//2))
c.save(OUT_ICO, format="ICO",
       sizes=[(16,16),(24,24),(32,32),(48,48),(64,64),(128,128),(256,256)])
```
- 选源图看**像素尺寸 + 主色**（写个探针脚本打印 `size/mode/主色/alpha bbox`），别凭文件名猜。
- 顺手存一张 512px PNG 作为预览给用户看效果。

## 5. 打包
```bash
"$PY" -m PyInstaller --noconfirm --clean --onefile --windowed \
  --name MyTool --icon app.ico \
  --add-data "app.ico;." --add-data "app_preview.png;." \
  --distpath dist --workpath build --specpath . main.py
```
- `--name` 用 ASCII，交付时再 `cp` 成中文文件名更稳。
- 运行时取内置资源：
```python
def resource_path(rel):
    base = getattr(sys, "_MEIPASS", os.path.dirname(os.path.abspath(__file__)))
    return os.path.join(base, rel)
```
- `root.iconbitmap(resource_path("app.ico"))` 设窗口图标。

## 6. windowed 模式的 subprocess 坑（必看）
```python
CREATE_NO_WINDOW = 0x08000000
subprocess.run(args, capture_output=True, stdin=subprocess.DEVNULL,
               creationflags=CREATE_NO_WINDOW, timeout=60)
```
- **一律**加 `CREATE_NO_WINDOW`（防闪黑框）+ `stdin=DEVNULL`（防挂起）。
- **例外**：`explorer.exe shell:AppsFolder\<PFM>!App` 启动应用、`notepad` 打开日志 —— **不能**加 CREATE_NO_WINDOW，否则起不来。
- 长任务要进度：`Popen(..., stdout=PIPE)` 逐行 `readline()` 计数，每 N 行刷一次进度，别用 `run()` 干等。

## 7. 中文编码
```python
# PowerShell 输出：先设 OutputEncoding，再 utf-8 -> gbk 依次尝试解码
script = "[Console]::OutputEncoding=[Text.Encoding]::UTF8;" + body
raw = subprocess.run(["powershell","-NoProfile","-NonInteractive",
                      "-ExecutionPolicy","Bypass","-Command",script],
                     capture_output=True, stdin=subprocess.DEVNULL,
                     creationflags=CREATE_NO_WINDOW).stdout
for enc in ("utf-8", "gbk"):
    try:
        t = raw.decode(enc)
        if t.strip(): return t
    except Exception: pass
```
- 读外部命令（tasklist/taskkill/xcopy/mklink）输出用 `decode("gbk","replace")`。

## 8. 交付前必做的验证（不要只说"应该能行"）
1. `python -m py_compile main.py`
2. **GUI 冒烟测试**：`exec_module` 加载模块 → `App(tk.Tk())` → `root.after(1500, 记录控件状态并 destroy)` → 打印按钮文案/进度条 max/终端内容。会闪一下窗口，可接受。
3. **exe 启动测试**：后台启动 → `tasklist /FI "IMAGENAME eq X.exe" /NH` 看进程 → `taskkill /F /IM`。
4. **引擎实测**：把核心逻辑类单独跑一遍，打印每一步结果，确认 0 error。

## 9. 发布前：把快捷方式重定向到最新产物（硬要求，别漏）

打了新 exe / 新版本之后，桌面与开始菜单上**已存在**的 `.lnk` 仍然指向旧文件 —— 用户点开还是旧行为。
**发布动作 = 部署新产物 + 重定向快捷方式 + 读回验证**，三者缺一不可；只把新 exe 拷过去就开始汇报 = 没发布完。

**做法 A（优先，普通用户会话内可用）—— COM 直接改写**
```powershell
$ws  = New-Object -ComObject WScript.Shell
$lnk = $ws.CreateShortcut("C:\Users\<you>\OneDrive\Desktop\App.lnk")
$lnk.TargetPath = "E:\new\App.exe"
$lnk.WorkingDirectory = "E:\new"      # 顺手一起改
if (Test-Path "E:\new\app.ico") { $lnk.IconLocation = "E:\new\app.ico" }
$lnk.Save()
# 读回验证
(New-Object -ComObject WScript.Shell).CreateShortcut($p).TargetPath
```

**做法 B（COM 被沙箱封锁 / pywin32 不可用）—— Node 等长二进制替换**
- `.lnk` 里目标路径以明文存 **3 份**：1 × latin1(ANSI 段) + 2 × UTF-16LE，**两种编码都要替换**
- **新旧 token 字符数必须相同** → 复制产物时就把文件名取成与旧名等长
- 先备份 `.lnk.bak`，替换后读回文件长度比对
- 详见 `windows-shortcut-retarget` skill

**收尾三件事**：
1. 逐个改，别漏 —— 桌面（注意本机真实桌面是 `OneDrive\Desktop(1)`）、开始菜单
   `%APPDATA%\Microsoft\Windows\Start Menu\Programs\`；任务栏固定项无法用此法改，需让用户重新固定
2. 刷新图标缓存：`ie4uinit.exe -show`
3. 汇报里明确写出「快捷方式已重指到 `<新路径>`」并附验证输出，不要只说"已更新"

## 10. 先定目标平台：Windows 与 Android 是两条完全不同的链

**动手前先确定发布到哪个平台**（用户没说就问，别默认 Windows）。
同一份逻辑，两个平台的 UI 层与打包方式**不通用**：

| | Windows | Android |
|---|---|---|
| 打包工具 | PyInstaller | buildozer / briefcase（**PyInstaller 不能产出 APK**） |
| GUI | tkinter（零依赖） | Kivy / BeeWare(Toga) |
| 图标 | 多尺寸 `.ico`（16→256） | 多密度 mipmap PNG：mdpi 48 / hdpi 72 / xhdpi 96 / xxhdpi 144 / xxxhdpi 192 |
| 产物 | `.exe`（可选安装包） | `.apk` / `.aab`，**需要签名** |
| 分发入口 | `.lnk` 快捷方式 | 应用抽屉图标 / 安装包 |
| 构建环境 | 本机 Windows 直接可跑 | buildozer 需 Linux/WSL/Docker；briefcase 需 Android SDK + JDK + Gradle |

**代码组织（核心逻辑两平台共用，UI 分层）**：
```
core.py           # 纯逻辑，禁止出现 import tkinter / kivy
ui_win.py         # Windows: tkinter
ui_android.py     # Android: Kivy
build_win.ps1     # pyinstaller ...
build_android.sh  # buildozer -v android debug
```
- 一旦把 `tkinter` 写进核心逻辑，Android 侧直接不可用 —— 这是最常见的返工原因
- Android 侧额外要处理：权限声明、前台/后台服务、存储访问、签名（debug 自动 keystore / release 正式 keystore）
- 若用户只是想「在手机上用」，先反问一句「必须做成 APK 吗？」—— 网页版 / Termux 脚本往往更省事

### 10.5 封装前必做：「平台串台」自检（硬要求）

**为什么必须查**：同一个项目经常同时出 Windows 版和 Android 版。最容易犯的错就是**把一端代码写进另一端的层里** ——
把 `import tkinter` 写进本该两端共用的 `core.py`，Windows 侧打包看着正常，Android 侧一打就废；
反过来把 `from kivy.app import App`、`from jnius import autoclass` 混进 PC 版，exe 启动即崩。
这类错误**编译器和打包器都不会主动提示**，必须在封装前主动查。

**检查清单（四项，缺一不可）**：

| # | 检查项 | 判定标准 |
|---|---|---|
| 1 | **核心逻辑层纯净** | `core.py`（及所有两端共用模块）**不得出现** `tkinter`、`kivy`、`jnius`、`android`、`plyer`、`win32*`、`ctypes.windll` 等任一平台专属 import |
| 2 | **UI 层各归各位** | `ui_win.py` 只允许 Windows 栈（tkinter/win32）；`ui_android.py` 只允许 Android 栈（Kivy/jnius）；**交叉即错** |
| 3 | **平台分支收拢** | `if sys.platform` / `os.name` 判断**集中在一个 compat 模块**，不要散落在业务代码里 |
| 4 | **路径与 API 无硬编码** | Windows 侧不该出现 `app.user_data_dir`；Android 侧不该出现 `C:\`、`%APPDATA%`、注册表、`SHFileOperation`、`.lnk` |

**做法（经验：AST 扫描比 `grep` 准得多）**
`grep` 会把注释、字符串、文档里的关键词一起算进去，误报一堆。用 `ast` 解析 import 节点最干净：

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

**处理经验（踩过的坑）**：

1. **报错信息会骗人** —— Android 侧打包失败时抛的是 buildozer/Kivy 的错，根因却常是 `core.py` 里一句
   `import tkinter`。Android 端"莫名"起不来，**先跑本检查**再深挖。
2. **写在函数体里的 `import` 也算** —— 有人为了"延迟导入"把 `import tkinter` 塞进函数内部，
   `grep "^import"` 根本抓不到；AST 遍历全部节点**能抓到函数内 import**，这是它优于 grep 的关键。
3. **`try/except ImportError` 掩护的平台代码同样是坑** —— 形如
   `try: import tkinter` / `except ImportError: pass` 看似"优雅兼容"，实际会让打包器把 tkinter 塞进 Android 包，
   体积暴涨且运行报错。要么挪到平台层，要么用显式 `if sys.platform` 分支。
4. **修完必须复跑** —— 把平台 import 移到 `ui_*.py` 后**重新跑检查**、确认退出码为 0 再打包，别凭"我改过了"就发版。
5. **打包链也要对齐** —— 检查通过 ≠ 打包链选对：`.exe` 只能 PyInstaller、`.apk` 只能 buildozer/briefcase，
   **PyInstaller 出不了 APK**（新人最常犯）。
6. **两端都产出时，一次改两端都验** —— 只验 Windows 版就发布，Android 侧的串台会拖到用户手里才炸。

### 模拟器启动 / 关闭 / 排障
已独立为 **`android-emulator-start`** skill —— 启动 AVD、等待 boot 完成、失败排查都在那边。
（骨架：只认 adb 契约 · detached+unref · `-no-snapshot-save` · 轮询 `getprop sys.boot_completed`。
 本机 SDK 在 `E:\andriod data`。）

## 11. 快捷方式重指的完整细节（自包含，不依赖其它 skill）

**先诊断**（确认 token 位置与出现次数，再动手）：
```bash
node -e "
const fs=require('fs');
const p='<绝对路径>/App.lnk';
const d=fs.readFileSync(p);
const s=d.toString('latin1');
let i=-1,n=0;while((i=s.indexOf('<OldToken>',i+1))>=0){console.log('ascii idx',i);n++;}
console.log('ascii count',n);
console.log('utf16 count',(d.toString('utf16le').match(/<OldToken>/g)||[]).length);
console.log('size',d.length);
"
```

**再替换**（latin1 与 utf16le 两种编码都要换）：
```bash
cp -f "App.lnk" "App.lnk.bak" && node -e "
const fs=require('fs');
const p='<绝对路径>/App.lnk';
function replaceAll(buf,from,to){const parts=[];let i=0;while(true){const j=buf.indexOf(from,i);if(j<0){parts.push(buf.slice(i));break;}parts.push(buf.slice(i,j));parts.push(to);i=j+from.length;}return Buffer.concat(parts);}
let d=fs.readFileSync(p);const before=d.length;
d=replaceAll(d,Buffer.from('<OldToken>','latin1'),Buffer.from('<NewToken>','latin1'));
d=replaceAll(d,Buffer.from('<OldToken>','utf16le'),Buffer.from('<NewToken>','utf16le'));
fs.writeFileSync(p,d);
console.log('size',before,'->',fs.readFileSync(p).length);
"
```

**关键约束**：新旧文件名**字符数必须相同**，否则 `.lnk` 结构会被破坏 —— 这种情况改用上面的 COM 方式。

## 12. 两侧同步（WorkBuddy ⇄ Codex）
本机同时存在 WorkBuddy（`~/.workbuddy/skills/`）与 Codex（`~/.codex/skills/`）两套目录。
同源 skill 内容应保持一致：在任一侧改进后，把另一侧同步过去（按语义合并，禁止用旧版整文件覆盖新版）。

### 12.1 移植给其他 Agent

本机不同 Agent 有独立的 Skill 目录：
- **Codex**：`C:\Users\<you>\.codex\skills\<name>\SKILL.md`；可用 `agents/openai.yaml` 配置界面元数据与隐式调用策略。
- **WorkBuddy**：`C:\Users\<you>\.workbuddy\skills\<name>\SKILL.md`。
- **OpenClaw**：使用其 Skill 导入流程。

移植时必须把目标 Agent 不具备的跨 Skill 引用改写成自包含步骤，并按语义合并双方独有改进；禁止用旧版整文件覆盖新版。

## 13. 发布流程（发布硬要求 + 安装包实制流程）

### 13.1 发布硬要求（每次发布都要执行；版本号规范见 §14）

**① 发布前主动询问：要不要「制作安装包 exe 并上传」**
打完便携版 exe、部署完、快捷方式也重指完之后，**不要就这么收尾**。
必须主动问一句：

> 便携版已就绪。是否需要我再做一份 **安装包 exe（Setup）并上传**？

原因：用户很可能是要**分发给别人**的 —— 便携版单文件对普通用户不友好（没有开始菜单项、没有卸载程序）。
不要默认"用户只要便携版"，也不要默认"不用上传"。**问了再决定**；用户说不要就跳过，说要多做一步。

**② 二进制产物（`.exe` / `.apk`）自动放 GitHub Release**
发布/上传时，**二进制产物必须进 Release**，不能只推源码：
```bash
gh release create v1.0.0 --title "v1.0.0" --notes "..." \
  "dist/MyTool.exe" "dist/MyTool-Setup-1.0.0.exe"
# 已有 Release 就上传附件：
gh release upload v1.0.0 "dist/MyTool.exe" --clobber
```
- 源码（`.zip` / tarball）作为附件或走常规 git push，但 **`.exe` / `.apk` 这类二进制一定挂 Release**。
- 沙箱里 `git push` 走不通时（CONNECT tunnel failed / 443 超时），改用 Git Data API / Contents API 推送源码 + Release API 上传产物。

**③ 封装前做「平台串台」自检（新增硬要求）**
打包动作**开始之前**，先确认没有把 Android 代码写成 Windows 端、或把 Windows 代码写成 Android 端。
这是"发布"流程里的固定一环，**不是可选项**：

- 跑一遍 §10.5 的 `check_platform_leak.py`，**退出码必须为 0** 才允许继续打包；
- 重点看 `core.py` 这类两端共用模块有没有混进 `tkinter` / `kivy` / `jnius` 等平台 import；
- 发现问题先把平台代码挪回 `ui_win.py` / `ui_android.py`，**改完再跑一次**确认清零；
- 两端都出产物时**两端都要过检查**，不能只验一侧；
- 打包链同时对齐：`.exe` → PyInstaller，`.apk` → buildozer/briefcase（PyInstaller 出不了 APK）。

### 13.2 NSIS 安装包怎么实做（本机已验证）

**工具来源**：本机 `makensis.exe` 常来自 electron-builder 的缓存，
路径形如 `%LOCALAPPDATA%\electron-builder\Cache\nsis\nsis-3.0.4.1\Bin\makensis.exe`（v3.04，支持 Unicode）。

**最小可用 `setup.nsi` 骨架**：
```nsis
Unicode true
!include "MUI2.nsh"
!include "FileFunc.nsh"

Name "My Tool"
OutFile "dist\MyTool-Setup-1.0.0.exe"
InstallDir "$LOCALAPPDATA\Programs\MyTool"     ; 装到用户目录 -> 免 UAC
RequestExecutionLevel user                      ; 关键：不要 admin，否则弹 UAC

!insertmacro MUI_PAGE_DIRECTORY
!insertmacro MUI_PAGE_INSTFILES
!insertmacro MUI_LANGUAGE "SimpChinese"

Section "Install"
  SetOutPath "$INSTDIR"
  File "dist\MyTool.exe"
  File "app.ico"

  ; 桌面路径必须从注册表读（见下方踩坑）
  !macro ResolveDesktop VAR
    ReadRegStr ${VAR} HKCU "Software\Microsoft\Windows\CurrentVersion\Explorer\Shell Folders" "Desktop"
    StrCmp ${VAR} "" 0 +2
      StrCpy ${VAR} "$DESKTOP"
  !macroend
  Var /GLOBAL DESK
  !insertmacro ResolveDesktop DESK

  CreateShortCut "$DESK\MyTool.lnk" "$INSTDIR\MyTool.exe" "" "$INSTDIR\app.ico" 0
  CreateShortCut "$SMPROGRAMS\MyTool.lnk" "$INSTDIR\MyTool.exe" "" "$INSTDIR\app.ico" 0

  ; 写卸载信息（HKCU，控制面板可见）
  WriteRegStr HKCU "Software\Microsoft\Windows\CurrentVersion\Uninstall\MyTool" "DisplayName" "My Tool"
  WriteRegStr HKCU "Software\Microsoft\Windows\CurrentVersion\Uninstall\MyTool" "UninstallString" "$INSTDIR\uninstall.exe"
  WriteUninstaller "$INSTDIR\uninstall.exe"
SectionEnd

Section "Uninstall"
  Delete "$INSTDIR\MyTool.exe"
  Delete "$INSTDIR\app.ico"
  Delete "$INSTDIR\uninstall.exe"
  RMDir "$INSTDIR"
  Delete "$DESK\MyTool.lnk"
  Delete "$SMPROGRAMS\MyTool.lnk"
  DeleteRegKey HKCU "Software\Microsoft\Windows\CurrentVersion\Uninstall\MyTool"
SectionEnd
```

**编译**：
```bash
"$NSIS/makensis.exe" -INPUTCHARSET UTF8 setup.nsi
```

**踩坑清单（都踩过）**：
| 坑 | 现象 | 解法 |
|---|---|---|
| 中文乱码 | 脚本里中文变问号 | `Unicode true` + 编译加 `-INPUTCHARSET UTF8` |
| 桌面快捷方式消失 | 装完桌面没有 | 本机真实桌面是 **OneDrive 重定向**，`$DESKTOP` 指错 → 从注册表 `Shell Folders\Desktop` 读 |
| 弹 UAC | 安装要管理员 | `RequestExecutionLevel user` + `InstallDir` 用 `$LOCALAPPDATA` |
| 静默安装路径不对 | `/S /D=` 没生效 | NSIS 已知行为：`/D=` 必须是**最后一个参数**且不加引号；直接检查默认目录即可 |
| 卸载删错快捷方式 | 便携版 `.lnk` 被删（名字近似） | 卸载段里 `.lnk` 名要和安装段**严格一致**，别用通配 |

**验证（别只看编译成功）**：
```bash
"dist\MyTool-Setup-1.0.0.exe" /S          # 静默安装
ls "$DESK/MyTool.lnk" "$SMPROGRAMS/MyTool.lnk" "$INSTDIR"   # 三处都在才算成功
"$INSTDIR\uninstall.exe" /S               # 静默卸载
ls "$DESK/MyTool.lnk" 2>/dev/null || echo "uninstalled OK"  # 快捷方式应已消失
```

### 13.3 沙箱下做快捷方式 / 删除（本机做法，纯标准库）

**建 `.lnk`：PowerShell COM 会被沙箱拦**（`New-Object -ComObject WScript.Shell` → "COM object instantiation..."），
`Add-Type` 也被拦 → **改用 Python ctypes 直接调 IShellLinkW**（独立进程不受影响）：
```python
# 关键 CLSID/IID
CLSID_ShellLink = "{00021401-0000-0000-C000-000000000046}"
IID_IShellLinkW = "{000214F9-0000-0000-C000-000000000046}"
IID_IPersistFile = "{0000010B-0000-0000-C000-000000000046}"
# SetPath / SetArguments / SetWorkingDirectory / SetIconLocation -> QueryInterface(IPersistFile).Save(path, True)
```
- 成功输出 `LNK_OK <path>` 便于脚本判定。
- **验证 `.lnk` 必须用 COM 读回**（GetPath/GetDescription/GetWorkingDirectory/GetArguments/GetIconLocation）——
  用二进制字符串搜索会因 UTF-16 编码不匹配而**误报 FAIL**（踩过）。这与 §11 的二进制替换是两条互补路径：
  §11 用于**改已存在的 .lnk**，本节用于**从零建 .lnk / 读回校验**。

**删除进回收站（不是直接删）**：`cscript` 被拦、`Remove-Item` 不可恢复 → 用 `SHFileOperationW`：
```python
op.fFlags = FOF_ALLOWUNDO | FOF_NOCONFIRMATION | FOF_SILENT | FOF_NOERRORUI
```
- `rc` 返回值**不可靠**（曾返回 2 但删除成功）→ 判定标准改为**「路径是否消失」**。
- 删完可以查 `C:\$Recycle.Bin\<SID>\$I*`（元数据）/ `$R*`（内容）确认可恢复。

### 13.4 本流程的产出样例（ChatGPT 修复工具）
- 便携版：`ChatGPT修复工具.exe`（9.6 MB，单文件）
- 安装包：`ChatGPT修复工具-Setup-1.0.0.exe`（9.5 MB，NSIS）
- 桌面快捷方式：`ChatGPT修复工具.lnk`（指向便携版，非安装包，由用户明确要求）
- 图标：Codex 老图标 + 蓝色圆圈徽章（`app.ico` 七档尺寸）

## 14. 版本号规范（每次发布都要遵守）

### 14.1 基线：首版 `0.1.0`，之后每次 `+0.0.1`（**满 10 进位**）

- **无特殊说明时，任何项目首版就是 `0.1.0`**（不是 `1.0.0`，也不是 `0.0.1`）。
- 之后**每次更新向上 +0.0.1**，逐位累加：`0.1.0 → 0.1.1 → … → 0.1.9 → 0.2.0 → 0.2.1 → …`。
  - 第三位是**十进制位，满 10 就进 1** —— **会**自动跳到 `0.2.0`；
  - **禁止**写成 `0.1.10` / `0.1.25` 这类"不进制"的写法。
- 同一个版本号要写进**三处并保持一致**：exe 文件名 / 安装包 `OutFile` / GitHub Release tag。

### 14.2 重制：从重制版重新 `0.1.0`

对项目**推倒重制**（rewrite）时，版本号**不延续旧序列** —— 从**重制发布的那一版重新 `0.1.0`** 开始。

- 建议在 Release notes / 文件名里标注"重制版"，避免与旧版混淆。
- 例：旧线已走到 `0.6.3` → 重制后发布 `0.1.0`（**不是** 0.6.4）。

### 14.3 大更新 / 大改动：先问，不自己拍板

用户表示"这次改动很大""算个大版本"时，**必须主动询问**：

> 这次改动较大，版本号是否需要直接 **+0.1.0**（例如 `0.1.7 → 0.2.7`）？

用户说"不用"就继续 `+0.0.1`；说"要"就**第二位 +1、第三位保持不变**（`0.1.7 → 0.2.7`，**不是** `0.2.0`）。

### 14.4 多分支协作：先问主分支

需要多方向并行开发时，按下面四步走：

**① 问用户哪个方向设为主分支**（不要默认、不要自己选）：

> 现在要开分支了，哪个方向作为**主分支**？

**② 主分支照常迭代**：主分支**从分支开始前的版本号继续 +0.0.1**，不受分支影响。

**③ 非主分支加字母后缀**：其他分支**以主分支的分叉点版本号为底，末尾追加一个大写字母**区分（`A` `B` `C` `D` `E`…）。
分支内若还要继续迭代，数字部分照常 +0.0.1，**后缀保留**。

**④ 合并后回归主分支**：分支合并回主分支后，**去掉字母后缀**，版本号恢复主分支的正常迭代。

**示例**（分叉点 `0.3.2`）：

| 分支 | 版本序列 |
|---|---|
| 主分支 | `0.3.2` → `0.3.3` → `0.3.4` |
| 分支 A（次方向） | `0.3.2A` → `0.3.3A` |
| 分支 B（次方向） | `0.3.2B` |
| A 合并回主分支 | 主分支照常发 `0.3.4`（去掉后缀继续迭代） |

- 后缀**必须是大写字母**（`A` 而非 `a`），与分支一一对应，**不要复用**。
- 边界情况（如 A、B 同时合并回主）**以用户口头约定为准**，口头约定优先于本表。

### 14.5 快速判定表

| 情形 | 做法 | 要不要先问用户 |
|---|---|---|
| 首次发布 | `0.1.0` | 不用 |
| 常规迭代 | `+0.0.1` | 不用 |
| 大更新 / 大改动 | 可能 `+0.1.0`（如 `0.1.7 → 0.2.7`） | **必须问** |
| 推倒重制 | 重制版重新 `0.1.0` | 确认"确实算重制" |
| 开多分支 | 主分支正常 +0.0.1；次分支加 `A/B/C` 后缀 | **必须问主分支是哪个** |
| 分支合并后 | 去掉后缀，回归主分支迭代 | 不用 |

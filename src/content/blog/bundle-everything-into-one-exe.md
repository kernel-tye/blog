---
title: '把一整套运行环境塞进一个安装包'
description: '目标机器在国内、网络受限、可能连 pip 都装不上东西。于是决定把 Python、Chromium、Node 全带上，做成一个双击就完事的 exe。1 GB 的原始素材，四个坑，和一个删掉 271 MB 的判断。'
pubDate: 'Aug 3 2026'
heroImage: '../../assets/blog-placeholder-4.jpg'
---

我给自己做了个命令行工具，附带一套 Python 脚本。在我这台机器上跑得好好的：Python 有、依赖装过、浏览器自动化用的 Chromium 也在。

问题是它要装到国内另一台机器上，**那台机器 pip 大概率超时，npm 更别想**。让用它的人先去配一遍环境，等于这工具没做完。

所以决定：**全带上**。做一个双击就完事的安装包，装完断网也能用。

素材总量 1 GB。下面是把它变成一个 exe 的过程，以及路上的四个坑。

## 为什么主程序是三个 exe

先说个不是坑但容易踩的事。我的工具编出来是三个可执行文件，**必须放在同一个目录**，分开放会报 `program not found`。

原因是 Windows 的权限是**进程级**的：主程序用普通权限运行，创建沙箱账户要管理员权限，实际执行被隔离的命令又要降权到受限账户。三种权限没法在一个进程里切换，只能拆成三个二进制，主程序按同级路径去找另外两个。

打包时这决定了一件事：**这三个必须作为一个整体安装**，不能让用户选装其中一个。

## 坑一：便携 Python 默认不认第三方包

Python 官方有个 **embeddable** 发行版，解压即用、不写注册表、不改 PATH，正好适合塞进别的程序里。20 MB 出头。

但它开箱是残废的：**`site` 机制被关掉了**，装什么包都 import 不到。解压后目录里有个 `python312._pth`，长这样：

```
python312.zip
.

# Uncomment to run site.main() automatically
#import site
```

要把最后一行的注释去掉，再加一行 `Lib\site-packages`：

```powershell
(Get-Content python312._pth) -replace '^#\s*import site','import site' | Set-Content python312._pth
Add-Content python312._pth "Lib\site-packages"
```

它也不带 pip，得自己引：

```powershell
Invoke-WebRequest https://bootstrap.pypa.io/get-pip.py -OutFile get-pip.py
.\python.exe get-pip.py
```

到这一步，它才是个能用的 Python。

## 坑二：pip 会告诉你"已经装好了"，然后什么都不做

这个坑最阴。装依赖的时候，输出里有几行：

```
Requirement already satisfied: pyee<14,>=13 in C:\Users\...\AppData\Roaming\Python\...
Requirement already satisfied: greenlet<4.0.0,>=3.1.1 in C:\Users\...\AppData\Roaming\Python\...
```

**pip 看到宿主机的用户目录里已经有这两个包，就跳过了。** 于是安装包里缺 `pyee` 和 `greenlet`——在我的机器上一切正常，因为它们从 Roaming 里被找到了；拷到别的机器，直接 import 失败。

而且这个错误**在打包机上永远复现不出来**。你测多少遍都是好的。

正确的装法要同时给两个约束：

```powershell
$env:PYTHONNOUSERSITE = "1"
.\python.exe -m pip install --no-user --target "Lib\site-packages" pymupdf playwright
```

`--target` 强制装到指定目录，`PYTHONNOUSERSITE` 让它别去看用户目录。

装完必须验证，而且**不能只验证 import 成功**——要看模块的真实路径：

```powershell
.\python.exe -c "import fitz,playwright,pyee,greenlet; [print(m.__file__) for m in (fitz,playwright,pyee,greenlet)]"
```

四个路径都必须落在你的便携目录下。有一个跑到 `AppData\Roaming` 里，这个包就是坏的。

## 坑三：Chromium 有 271 MB 是白带的

浏览器自动化用的 Chromium，默认下到 `%LOCALAPPDATA%\ms-playwright`。打包时用环境变量把它引到包内：

```powershell
$env:PLAYWRIGHT_BROWSERS_PATH = "<绝对路径>\runtime\ms-playwright"
.\runtime\python\python.exe -m playwright install chromium
```

拉下来一看，**701 MB**。展开：

| 目录 | 大小 |
| --- | --- |
| `chromium-*` | 427 MB |
| `chromium_headless_shell-*` | **271 MB** |
| `ffmpeg-*` | 3 MB |

`ffmpeg` 是录视频用的，跟我的场景无关，删。

**无头壳那 271 MB 才是有意思的部分。** 我本来当然想用无头模式——不弹窗口、能做定时任务，正是自动化想要的。但实测下来：

- 一家网站给无头浏览器返回 Cloudflare 的「请稍候…」检查页
- 另一家直接返回 **HTTP 418**（I'm a teapot，这是它们的反爬响应）

同一个会话、同一份 cookie，换成有窗口的浏览器就一切正常。

后来发现还要再进一步：**Playwright 自带的 Chromium 也会被识别**，得用系统上装的真实 Chrome，并且关掉自动化标记：

```python
pw.chromium.launch_persistent_context(
    profile, channel="chrome", headless=False,
    args=["--disable-blink-features=AutomationControlled"],
    ignore_default_args=["--enable-automation"],
)
```

这么一来无头壳永远用不上，**删得心安理得**。701 → 427 MB。

顺带说，"无头不能用"这个结论反过来影响了整个功能的形态：它注定是"弹个窗口自己点"而不是"后台静默跑"，那么定时任务的设计就得跟着变。**一个打包时的体积决策，暴露的是产品形态的约束。**

## 坑四：别把便携 Python 加进 PATH

最直觉的做法是把 `runtime\python` 加到 PATH 里，这样脚本直接 `python xxx.py` 就能跑。

**千万别。** 目标机器上大概率有 conda 或者别的 Python，那是**别人的环境**，一个工具不该去改写它。加进 PATH 就意味着从此那台机器上所有的 `python` 都变成了你的。

正确做法是给一个显式入口，一个 `.cmd` 就够：

```cmd
@echo off
setlocal
set "PLAYWRIGHT_BROWSERS_PATH=%~dp0ms-playwright"
set "PYTHONNOUSERSITE=1"
set "PYTHONIOENCODING=utf-8"
set "PYTHONUTF8=1"
"%~dp0python\python.exe" %*
endlocal
```

五行，每一行都对应一个具体的坑：

| 变量 | 不设会怎样 |
| --- | --- |
| `PLAYWRIGHT_BROWSERS_PATH` | 去用户目录找浏览器，找不到就**联网重下 400 MB**——而这套东西正是为了离线 |
| `PYTHONNOUSERSITE` | 宿主机的包混进来，就是坑二的运行时版本 |
| `PYTHONIOENCODING` / `PYTHONUTF8` | 中文路径下打印会抛 `UnicodeEncodeError` |

最后一条我是被 conda 教育的。工作目录叫 `新建文件夹 (8)`，conda 激活时要打印这个路径，默认编码是 cp1252，编不出中文，**它自己的异常处理器跟着炸**，吐出一大片红色 ERROR REPORT 加终端响铃。看起来像程序崩溃，实际上只是输出编码——底下的命令照常跑完了。

## 安装器：Inno Setup

打包工具用 Inno Setup，免费、单文件输出、自带卸载器。

```powershell
winget install JRSoftware.InnoSetup
& "$env:LOCALAPPDATA\Programs\Inno Setup 6\ISCC.exe" installer.iss
```

注意 winget 装的位置是 `%LOCALAPPDATA%\Programs\`，不在 `Program Files`——那是免管理员安装。我第一次按惯例去 `Program Files` 找，没找到。

### 我写错的两处

**`AppId` 得是合法 GUID。** 我图省事写了 `{{7A3C9E12-...-TYCODE0001}}`——最后一段必须是十六进制，`TYCODE` 里的 T、Y、C、O 都不是。

顺带一提，**这个值定下来就不要改**。它是安装器识别"同一个程序"的依据，改了之后新版本会被当成另一个软件，旧的卸不掉，两个条目并存。

**`{userappdata}` 不是用户目录。** 它指向 `C:\Users\你\AppData\Roaming`，我写了 `{userappdata}...xxx` 想上一级，结果上到了 `AppData`，文件会装到 `C:\Users\你\AppData\.tycode`。要用 `{%USERPROFILE}.xxx`。

### 卸载时要清的，不在安装目录里

这一条是我这个工具特有的，但思路通用：**你的程序在系统里留下的东西，删文件夹是删不掉的。**

我的工具运行时会创建：

- 本地用户账户（跑沙箱用的受限账户）
- 一个本地组
- 三条防火墙规则

卸载时得显式清理：

```
[UninstallRun]
Filename: "powershell.exe"; RunOnceId: "RemoveSandboxUsers"; \
    Parameters: "-NoProfile -Command ""Get-LocalUser -Name 'XxxSandbox*' | Remove-LocalUser"""
Filename: "netsh.exe"; RunOnceId: "RemoveFirewallRules"; \
    Parameters: "advfirewall firewall delete rule name=all program=""{app}\xxx.exe"""
```

值得盘一遍你的程序装完之后在系统里留下了什么：注册表项、计划任务、服务、账户、防火墙规则、证书。**只有文件是删文件夹能带走的。**

### 用户数据要留下

`~/.xxx` 里有配置、凭据、会话历史和用户自己改过的脚本。删掉是不可逆的，重装还要重新配。所以默认保留，只在卸载时弹一次询问，**默认按钮是"否"**。

程序装在 `Program Files`，用户数据在用户目录——这条边界也决定了哪些跟着卸载走、哪些留下。

## 凭据单独成一个组件

我这个包里带了自己的 API key，因为纯自用，装完就能跑。但这意味着**这个 exe 不能给别人**——谁拿到谁就能用我的额度。

处理方式是把凭据做成一个可选组件，并且给三种安装类型：

```
完整安装（含我的配置）   ← 给自己的机器
仅程序与技能            ← 给别人，不带凭据
自定义
```

源文件目录写进 `.gitignore`，安装包输出目录也写进去。`installer.iss` 本身不含密钥，可以照常提交。

另外凭据文件用 `onlyifdoesntexist` 标志——**已有配置绝不覆盖**。升级版本不会把目标机上已经配好的东西冲掉。

## 结果

```
原始素材    1005 MB
安装包       272 MB      压缩率 27%
压缩耗时    5 分 36 秒
```

用的是 `Compression=lzma2/max` 加 `SolidCompression=yes`。后者是"固实压缩"——把所有文件当成一个连续数据流压，跨文件的重复能被利用到。Chromium 和 Python 里有大量相似的二进制片段，这个开关值不少。

代价是压缩过程慢（这次五分半），而且安装时不能只解压其中一个文件——但安装本来就是全量的，无所谓。

装完的结构：

```
Program Files\XxxCode\
  三个 exe
  runtime\
    xxx-python.cmd      ← 显式入口，不污染 PATH
    python\             Python + 依赖
    ms-playwright\      Chromium（无头壳已删）
    node\               Node LTS
  unins000.exe

用户目录\.xxx\
  skills\               脚本，卸载时保留
  config.toml           只在选了凭据组件时写，且不覆盖
```

## 值得带走的几条

1. **在打包机上验证便携环境是无效的**——它总能从宿主机借到缺的东西。要验证依赖的**真实路径**，不是 import 成功与否。
2. **体积决策会暴露产品约束**。我删掉 271 MB 的无头浏览器，不是因为省空间，是因为发现无头模式根本用不了——而这直接决定了这个功能不可能做成静默的定时任务。
3. **别动别人的 PATH**。多一个 `.cmd` 入口的代价，远小于把目标机上的 `python` 改掉。
4. **卸载要清的东西比你以为的多**。文件之外还有账户、规则、注册表；而用户数据反过来不该清。

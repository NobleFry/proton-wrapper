# proton-wrapper

一直以来，用 Proton 运行 exe 大多通过添加非 Steam 游戏或者Lutris之类的方式，难以做到`双击直接运行`般无缝的交互体验，所以我制作了 `shorin-proton-wrapper`。最大的特点是安装后可以直接运行 exe，无须任何前置操作。

Arch Linux 可以从AUR安装。

>注意，这只是个包装器，原本 Proton 运行不了的软件依旧会运行不了。

![](./pics/1.png)

灵感来自 Wine 的配置过程。安装 Wine 后运行 `winecfg` 会在 `~/.wine` 目录初始化前置运行环境。我借鉴并改良了这个设计，运行 exe 的默认行为是检测 `~/.proton` 是否存在，如果不存在则静默初始化。默认运行器是 [`WE-Proton`](https://github.com/SHORiN-KiWATA/we-proton)：它基于 `GE-Proton` 和 `DW-Proton`，额外修了一些 Wine 的 bug，能运行 WeGame 和一些国服游戏，还内置了游戏加速器 TUN 模式需要的 wintun（见下面的「独立网络」）。

运行器默认装在 `~/.local/share/Steam/compatibilitytools.d`，Steam 和不少第三方启动器也读这个目录。这里（或旧版本用的 `~/.local/share/proton/runners`、Lutris 的目录）已经有 `WE-Proton Latest`、`Proton-GE Latest`、`DW-Proton Latest` 时直接使用，不会重复下载；`--update` 会一并更新这几个位置里的 Latest 运行器。

首次自动安装运行器时，临时下载文件会放在 `~/.cache/shorin-proton-wrapper/downloads`，安装成功后清理。下载、解压、初始化过程中只会显示一条桌面通知并原地更新内容；无通知环境时自动降级为仅输出日志。下载不限总时长，只在连续 120 秒几乎没有流量时判定为卡死并中止；中断后重新运行会从断点续传。

安装失败时会弹出常驻通知说明原因，完整日志在 `~/.cache/shorin-proton-wrapper/logs/install-*.log`，前置环境初始化日志在 `~/.cache/shorin-proton-wrapper/logs/init.log`。

Proton 会放进它要求的 Steam Linux Runtime 容器里运行（WE-Proton、DW-Proton 和 GE-Proton 11 要 SLR 4.0，GE-Proton 10 要 sniper），否则自带的 GStreamer 插件等会缺库，视频和音频解码之类的功能出问题。已装 umu 或 Steam 的运行时会直接复用，没有就从 Valve 下载到 `~/.local/share/shorin-proton-wrapper/runtime`。

32 位程序默认用 Wine 的新 WoW64 模式运行（`PROTON_USE_WOW64=1`），不需要安装 32 位系统库。个别 32 位程序有问题时，可以在它的配置里把环境变量设成 `PROTON_USE_WOW64=0`。

可调环境变量：`SHORIN_PROTON_WRAPPER_DOWNLOAD_STALL`（卡死判定秒数，默认 120）、`SHORIN_PROTON_WRAPPER_DOWNLOAD_TIMEOUT`（下载总时长上限秒数，默认 0 表示不限制）、`SHORIN_PROTON_WRAPPER_RUNTIME`（设为 0 时不用 Steam 运行时，直接在宿主机上运行 Proton）。

## 设置Windows程序运行环境

>`shorin-proton-wrapper-configure`

为了兼顾自由度，我额外做了一个名为 `设置Windows程序运行环境` 的打开方式。支持自定义前置环境目录、Proton 版本、MangoHud、GameScope 等常用设置项。

![](./pics/2.png)

![](./pics/3.png)

第一次设置之后会将设置项缓存至 `~/.cache/shorin-proton-wrapper`，以后使用 `运行Windows软件` 打开会自动读取缓存配置，如果再次用 `设置Windows软件运行环境` 打开，则会提示 `已保存的运行环境`，可自行决定要重新设置还是直接运行。

![](./pics/4.png)

设置项里还有「以 Wayland 运行」：使用 Wine 的 Wayland 驱动，开 Gamescope 时不生效。用 Gamescope 运行时，如果桌面在用 fcitx5，会自动让 fcitx5 也接管 Gamescope 的显示，在 Gamescope 里也能打中文。

## 同一个前置环境里同时运行多个程序

同一个前置环境里的程序共用一个 Wine 会话。前置环境里已经有程序在运行时，再打开别的程序会直接加入，不用等前面的程序退出。加入的程序沿用会话正在用的运行器和显示方式，和自己的设置不同时会弹出通知说明：

- 运行器不同时，改用会话正在用的那个（两个版本的 Wine 不能共用一个会话）；
- 会话以 X11 运行时，「以 Wayland 运行」和 Gamescope 这次不生效；会话以 Wayland 运行时，加入的程序也以 Wayland 运行；
- 会话在 Gamescope 里时，其他程序不能加入（第一个程序退出时 Gamescope 会一起关掉）。

管理器里的「关闭」只结束这个程序自己，同一前置环境里的其他程序不受影响。前置环境外面的程序按所在文件夹各自分配盘符，同时运行也互不干扰。

## 独立网络（游戏加速器）

在管理器的前置环境一栏打开「独立网络」（或者 `shorin-proton-wrapper -p 前置环境 --netns on`）后，这个前置环境的程序会运行在独立的网络命名空间里，不需要 root。把 Windows 版的游戏加速器（例如雷神加速器的 TUN 模式）和要加速的游戏装进同一个前置环境，先打开加速器、再打开游戏，加速器建立的虚拟网卡只接管这个前置环境里的程序，系统的其他网络不受影响。

- 需要安装 `passt`（提供 `pasta`），运行器用 WE-Proton（内置 wintun）；
- 加速器按进程名决定哪些连接走加速，所以游戏要和加速器在同一个前置环境里；
- 网卡沿用主机网卡的 MAC 地址；DNS 查询不走加速器的隧道，避免占满加速节点的连接数；托盘图标、通知这些 D-Bus 功能照常可用；
- 开关在这个前置环境的程序全部关闭后生效；
- 程序看到的网卡和用户与平时不同，带反作弊的游戏请谨慎使用。

## Steam 游戏

在 Steam 里把游戏的启动选项设为：

```
shorin-proton-wrapper %command%
```

游戏就不再用 Steam 给它建的前置环境，而是作为 `~/.proton` 里的程序运行（另一个前置环境用 `shorin-proton-wrapper -p 前置环境 %command%`）：和那里的其他程序共用一个会话、运行器和独立网络，比如让加速器接管 Steam 游戏。

- 游戏的「兼容性」里仍要选一个 Proton（建议 WE-Proton Latest），否则 Steam 不会启动 Windows 游戏；实际运行用的是前置环境的运行器；
- 游戏安装目录分配一个盘符，游戏按 Steam 启动时的工作目录运行；
- 存在 Windows 用户目录（AppData、文档）里的存档会在 `~/.proton` 里。Steam 云存档同步的是 Steam 自己的前置环境（`steamapps/compatdata/<AppID>`），这类存档不会跟着同步，已有的存档需要自己复制过来；
- 带反作弊的游戏多半不行；
- 游戏会出现在管理器的应用列表里，但需要 Steam 的游戏请从 Steam 启动。

## Shorin Proton Wrapper 管理器

>`shorin-proton-wrapper-manager`

每次运行 exe 都会自动进入管理器的仓库，通过 `icoextract` `python-pillow` 生成缩略图。支持导出程序到 Linux 应用菜单、重命名显示名称和 `.desktop` 名称、修改配置等功能。

![](./pics/5.png)

在进程卡死或者后台进程无法退出的时候可以在进程页面强制退出。

![](./pics/6.png)

## shorin-proton-wrapper

另外还提供了 CLI 工具方便从终端进行 Proton 管理、exe 运行等操作。

```
shorin-proton-wrapper 

用法:
  shorin-proton-wrapper [选项] app.exe [参数...]

选项:
  --init                     初始化，默认路径 ~/.proton
  -p, --prefix 目录          指定前置环境路径，若不存在则初始化
  --status                   显示前置环境信息
  --list-runner              列出已安装的运行器
  --runner 路径|ge|dw|we     指定运行器
  --install ge|dw|we         安装运行器，默认安装至 ~/.local/share/Steam/compatibilitytools.d
  --install-steam ge|dw|we   同 --install
  --install-lutris ge|dw|we  给 Lutris 安装运行器
  --download ge|dw|we        仅下载运行器，默认下载路径为 XDG 下载目录
  --list-version ge|dw|we    列出可使用的运行器版本
  --version 版本             指定特定版本，默认 latest
  --update                   升级所有已安装的 Latest 运行器
  --check-update             检查运行器是否有可用更新（不安装）
  --doctor                   检查 wrapper 运行环境并输出诊断信息
  --remove ge|dw|we          移除默认位置中已安装的运行器
  --remove-steam ge|dw|we    移除 Steam 中指定运行器，省略 ge|dw|we 则移除全部由 wrapper 安装的运行器
  --remove-lutris ge|dw|we   移除 Lutris 中指定运行器，省略 ge|dw|we 则移除全部由 wrapper 安装的运行器
  --gs, --gamescope          用 Gamescope 运行。默认全屏；需要自定义分辨率、帧率、FSR 或遇到显示兼容问题时使用
  --mangohud                 启用 MangoHud 性能叠加层
  --wayland                  以 Wayland 运行（Proton 运行器，开 Gamescope 时不生效）
  --netns on|off             给前置环境开启独立网络命名空间（加速器用），需要 passt
  --debug                    将完整调试日志写入 ~/.cache/shorin-proton-wrapper/logs
  --help-full                显示完整说明

示例:
  shorin-proton-wrapper --init
  shorin-proton-wrapper /path/to/app.exe
  shorin-proton-wrapper --runner /path/to/proton /path/to/app.exe
  shorin-proton-wrapper -p /path/to/prefix /path/to/app.exe
  shorin-proton-wrapper --install we
  shorin-proton-wrapper --install-steam we
  shorin-proton-wrapper --download ge --version GE-Proton10-34
  shorin-proton-wrapper --gamescope --res 1920x1080 --fps 30 --fsr /path/to/app.exe
  shorin-proton-wrapper --debug /path/to/app.exe
  shorin-proton-wrapper --list-version we
  shorin-proton-wrapper --doctor
  shorin-proton-wrapper --update
```
```
shorin-proton-wrapper --gamescope 

用法:
  shorin-proton-wrapper --gamescope [Gamescope 选项] app.exe [参数...]

Gamescope 选项:
  --windowed             窗口模式
  --fullscreen           全屏模式 (默认)
  --res 宽x高            设置游戏渲染分辨率，例如 1920x1080
  --fps 帧率             限制帧率
  --fsr                  启用 FSR 超分辨率
  --args 参数            追加自定义 Gamescope 参数
  --mangohud             启用 MangoHud 性能叠加层
  -h, --help             显示本帮助

示例:
  shorin-proton-wrapper --gamescope --windowed --res 1920x1080 --fps 30 --fsr --mangohud game.exe
```

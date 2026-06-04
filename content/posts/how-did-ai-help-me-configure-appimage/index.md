---
title: "AI 如何帮我配置 AppImage"
date: 2026-06-04T21:56:43+08:00
draft: false
tags: ["ai"]
comments: true
---
> AppImage 是一种 Linux 应用程序打包和分发格式，核心理念： one app = one file ，开发者不再为每种发行版逐一打包。

虽然早就了解过 AppImage，但每次下载应用时还是习惯找 `deb` 安装包。最近想在本地安装 LM Studio，官方只提供了 AppImage 文件。下载完成后，只需赋予可执行权限即可运行。但如果想让它像其他应用一样出现在系统启动器中，还需要手动创建 `.desktop` 文件并更新 Desktop Database。今天就来记录一下，我是如何在 AI 的帮助下完成这项配置的。

首先，如果自己做 `.desktop` 文件，可能会先去官网下载软件的图标，但是 AI 没有使用 `curl` 或其他工具从网络上下载，而是尝试分析 AppImage 文件本身，原因在于 AppImage 是一种自包含的格式，应用程序及其资源文件（图标、`.desktop` 文件等）实际上都已经打包在文件本身中了。下面是 AI 的构建流程：

1. 确定 AppImage 类型：通过查看文件头确认是 Type2 格式的 AppImage （ELF 可执行文件）
```bash
head -c 100 LM-Studio-0.4.15-2-x64.AppImage | xxd | head -3
00000000: 7f45 4c46 0201 0100 4149 0200 0000 0000  .ELF....AI......
00000010: 0200 3e00 0100 0000 9046 4000 0000 0000  ..>......F@.....
00000020: 4000 0000 0000 0000 e8d7 0200 0000 0000  @...............
```
其中 `7f 45 4c 46` 是 ELF 魔数（`\x7fELF`），`41 49` 是 AppImage 魔数（`AI`），`02 00` 则代表格式是 Type2 。

2. 获取 `squashfs` 偏移量：Type2 AppImage 的本质是在 ELF 可执行文件后面附加了一个 `squashfs` 文件系统。
```bash
./LM-Studio-0.4.15-2-x64.AppImage --appimage-offset
188392  # 偏移量
```
> `squashfs` 是一种只读压缩文件系统，广泛用于 Linux 中。特点是压缩率高、只读、体积小，支持常见压缩算法。

AppImage 内部嵌入的目录结构：
```text
squashfs-root/
├── AppRun                # 启动入口
└── usr/
    ├── bin/
    │   └── myapp         # 主程序
    ├── lib/
    │   └── *.so          # 依赖库
    └── share/
        ├── icons/
        └── *.desktop
```
在执行 AppImage 时，会先运行头部的 Runtime，完整执行流程如下：

{{< mermaid >}}
flowchart TD
    A["用户双击 AppImage 文件"] --> B{"检查文件头"}
    B -->|"ELF 魔数: 7f 45 4c 46"| C["识别为 ELF 可执行文件"]
    B -->|"AppImage 魔数: 41 49"| D["识别为 AppImage (Type2)"]
    C --> D
    D --> E["执行头部 Runtime"]
    E --> F["获取 squashfs 偏移量<br/>（--appimage-offset）"]
    F --> G{"FUSE 可用？"}
    G -->|"是"| H["通过 FUSE 挂载 squashfs<br/>（默认方式）"]
    G -->|"否"| I["提取 squashfs 到临时目录<br/>（回退方式）"]
    H --> J["执行 AppRun 启动入口"]
    I --> J
    J --> K["加载 squashfs-root 结构"]
    K --> L["usr/bin/myapp<br/>（主程序）"]
    K --> M["usr/lib/*.so<br/>（依赖库）"]
    K --> N["usr/share/icons<br/>（图标资源）"]
    K --> O[".desktop<br/>（桌面配置）"]
    L --> P["应用程序启动完成"]

    style A fill:#4a9eff,color:#fff
    style E fill:#ff9f43,color:#fff
    style H fill:#54a0ff,color:#fff
    style I fill:#ffa502,color:#fff
    style J fill:#ff6348,color:#fff
    style P fill:#2ed573,color:#fff
{{< /mermaid >}}

3. 查看 `squashfs` 中的文件（定位图标文件）
```bash
unsquashfs -o 188392 -l LM-Studio-0.4.15-2-x64.AppImage | grep -iE '\.(png|svg)'
unsquashfs -o 188392 -ll LM-Studio-0.4.15-2-x64.AppImage 2>&1 | grep 'lm-studio.desktop'  # 这部分是我加的，看看
```

4. 提取图标
```bash
unsquashfs -o 188392 -d /tmp/lmstudio_squash \
    -f LM-Studio-0.4.15-2-x64.AppImage \
    usr/share/icons/hicolor/0x0/apps/lm-studio.png
```
内部 `lm-studio.desktop` 中 `Exec = AppRun --no-sandbox %U`，可执行文件指向 AppImage 内部相对路径，并不能拿来直接使用。

5. 创建 `.desktop` 文件、安装与卸载脚本
```desktop
[Desktop Entry]
Name=LM Studio
Comment=Run local LLMs like a pro
Exec=/home/yongy/Software/LM-Studio/LM-Studio-0.4.15-2-x64.AppImage --no-sandbox %U
Icon=/home/yongy/Software/LM-Studio/lm-studio.png
Type=Application
Categories=Development;IDE;ArtificialIntelligence;
StartupNotify=true
StartupWMClass=LM Studio
MimeType=x-scheme-handler/lmstudio;
Keywords=LLM;AI;Chat;Model;Local;
```
```sh
#!/bin/bash

SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

cp "$SCRIPT_DIR/lm-studio.desktop" ~/.local/share/applications/lm-studio.desktop

update-desktop-database ~/.local/share/applications/ 2>/dev/null

echo "Completed!"
```
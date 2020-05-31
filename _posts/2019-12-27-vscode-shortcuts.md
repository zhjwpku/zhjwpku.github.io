---
layout: post
title: "VS Code Shortcuts"
date: 2019-12-27 22:00:00 +0800
tags:
- hotkey
---

终于，又换编辑器了，之前就听人说 VS Code 特别好用，但由于懒，一直没有尝试。最近公司在推广 VSCode-huawei，大概就是在开源 vscode 的基础上去掉微软专有的插件，再提供一个内部的插件市场。笔者也借此机会，把家里（之前用的 sublime text 3）和公司（之前用 sublime text 3 和 source insight 4）的编辑工具统一一下。本文仅仅是 VS Code 常用快捷键及扩展插件的列举。

<h4>Shortcuts</h4>

```
                                    Mac                     Windows

## 通用

打开 shortcuts 网页文档           cmd K + cmd R            Ctrl+K Ctrl+R
在 vscode 显示 shortcuts         cmd K + cmd S            Ctrl+K Ctrl+S
显示命令面板                      cmd + shift + P          Ctrl+Shift+P, F1
跳转到文件及一些其他操作            cmd P                    Ctrl+P
    - 直接输入文件名，跳转到文件
    - ?  列出当前可执行的动作
    - !  显示 Errors或 Warnings
    - :  跳转到行数
    - @  跳转到当前文件的 symbol（搜索变量或者函数）
    - @: 根据分类跳转到当前文件的 symbol（搜索变量或者函数）
    - #  根据名字查找 symbol
切换 sidebar 显示                cmd B                    Ctrl+B
切换 Zen Mode 模式               cmd + K + Z              Control+K+Z
显示/隐藏 Panel                  cmd + J                  Control+J
Split Editor                    cmd + \                  Control+\
Focus on 1st Editor Group       cmd + 1                  Contrpl+1

## 跳转命令

跳转到文件                        cmd P                    Ctrl+P
跳转到行号                        control+G                Ctrl+G
跳转到当前文件的的符号              cmd + shift + O          Ctrl+shift+O
根据分类跳转到当前文件的符号         cmd + shift + O 并输入:   Ctrl+shift+O 并输入:
Go Back                         cmd + left               Ctrl+left
Go Forward                      cmd + right              Ctrl+right
Go to Definition                F12                      F12
Go to Implementations           cmd F12                  Ctrl+F12
Go to References                shift + F12              Shift+F12
Go to Bracket                   cmd + shift + \

## Bookmarks
Mark/Unmark                     cmd + option + K
Jump to Next                    cmd + option + L
Jump to Previous                cmd + option + J

## Build && RUN
Build                           cmd + shift + B
Run Test Task                   cmd + control + T

## 其它

预览 markdown 文档                 cmd + shift + V
打开当前文件所在的目录               cmd K + R                Ctrl K + R

```

<h4>Extensions</h4>

VS Code 提供了一个[插件市场][marketplace]，这里有很多强大的各式各样的插件，每个人可以根据开发语言及个人喜好选择自己的工具。

**General**

- [GitLens — Git supercharged][gitlens]
- [vscode-icons][vscode-icons]
- [VSCode Great Icons][vscode-great-icons]
- [scrollkey][scrollkey]
- [Remote - SSH][remote-ssh]
- [Bookmarks][bookmarks]
- [Bracket Pair Colorizer 2][bracket-pair-colorizer]
- [indent-rainbow][indent-rainbow]
- [shellcheck][shellcheck]
- [Copyright Inserter][copyright-inserter] *安装好之后修改源码可定制自己需要的 Copyright 声明*

**C/C++**

- [C/C++][ccpp]
- [CMake Tools][cmake-tool]

**Front-End**

- [Live Server][live-server]
- [REST Client][rest-client]

<h4>Settings</h4>

**settings.json**

```
{
    "workbench.iconTheme": "vscode-icons",
    "vsicons.dontShowNewVersionMessage": true,
    "terminal.integrated.shell.osx": "/bin/zsh",
    "editor.suggestSelection": "first",
    "vsintellicode.modify.editor.suggestSelection": "automaticallyOverrodeDefaultValue",
    "C_Cpp.updateChannel": "Insiders",
    "editor.fontSize": 14,
    "editor.rulers": [80, 100, 120],
    "cmake.configureOnOpen": false,
    "terminal.integrated.fontSize": 14,
    "debug.console.fontSize": 14,
    "git.ignoreLegacyWarning": true,
    "editor.renderWhitespace": "all",
    "window.zoomLevel": 0,
    "scrollkey.line1": 1,
    "scrollkey.line2": 10,
    "scrollkey.line3": 20,
    "files.autoGuessEncoding": true,
    "files.trimTrailingWhitespace": true,
    "files.autoSave": "afterDelay",
    "C_Cpp.default.cppStandard": "c++11",
    "editor.minimap.enabled": false,
    "zenMode.hideLineNumbers": false,
    "explorer.openEditors.visible": 0
}
```

**也可在每个 workspace 设置 settings.json**

*.vscode/settings.json*

```
{
    // 暗淡不活跃的代码区，如 #if 0/#endif 之间的代码
    "C_Cpp.dimInactiveRegions": false
}
```

**keybinds.json**

```
// Place your key bindings in this file to override the defaults
[
    {
        "key": "ctrl+cmd+e",
        "command": "workbench.action.editor.changeEncoding",
        "when": "editorTextFocus"
    },
    {
        "key": "cmd+up",
        "command": "scrollkey.up2",
        "when": "editorTextFocus"
    },
    {
        "key": "cmd+down",
        "command": "scrollkey.down2",
        "when": "editorTextFocus"
    },
    {
        "key": "ctrl+cmd+up",
        "command": "cursorTop",
        "when": "textInputFocus"
    },
    {
        "key": "ctrl+cmd+down",
        "command": "cursorBottom",
        "when": "textInputFocus"
    },
    {
        "key": "cmd+left",
        "command": "workbench.action.navigateBack"
    },
    {
        "key": "cmd+right",
        "command": "workbench.action.navigateForward"
    },
    {
        "key": "ctrl+cmd+t",
        "command": "workbench.action.tasks.test"
    },
]
```

**Code formatting**

 C/C++ 插件集成了 clang-format，默认情况下，clang-format 会在工作目录寻找 .clang-format 文件，如果找不到该文件，会使用 *Visual Studio* 的代码风格。clang-format 书写方式见 **[clang-format style options][ClangFormatStyleOptions]**。

**Sample tasks.json**

```
{
    // See https://go.microsoft.com/fwlink/?LinkId=733558
    // for the documentation about the tasks.json format
    "version": "2.0.0",
    "tasks": [
        {
            "type": "shell",
            "label": "build",
            "options": {
                "env": {
                    "CPLUS_INCLUDE_PATH": "/usr/local/include",
                    "LIBRARY_PATH": "/usr/local/lib"
                }
            },
            "command": "rm -rf build && mkdir build && cd build; cmake ..; make",
            "group": {
                "kind": "build",
                "isDefault": true
            },
            "problemMatcher": []
        },
        {
            "type": "shell",
            "label": "run",
            "command": "build/unit_tests",
            "dependsOn":["build"],
            "group": {
                "kind": "test",
                "isDefault": true
            },
            "problemMatcher": []
        }
    ]
}

```

**Sample launch.json**

```
{
    // Use IntelliSense to learn about possible attributes.
    // Hover to view descriptions of existing attributes.
    // For more information, visit: https://go.microsoft.com/fwlink/?linkid=830387
    "version": "0.2.0",
    "configurations": [
        {
            "name": "(lldb) Launch",
            "type": "cppdbg",
            "request": "launch",
            "preLaunchTask": "build",
            "program": "${workspaceFolder}/build/unit_tests",
            "args": [],
            "stopAtEntry": false,
            "cwd": "${workspaceFolder}",
            "environment": [],
            "externalConsole": false,
            "MIMode": "lldb"
        }
    ]
}
```

**Tips**

- 如果无法直接在vscode安装插件，可以获取相应的 vsix 文件，并使用 `Install from VSIX` 的方式安装插接件。

- 打开一个新文件会被另一个新打开的文件替换，是因为前一个文件是一个不固定的 Editor（可以通过 Editor 上的字体来判断，斜体表示没有固定，正体表示固定），可以使用`双击 Editor `或者`双击 sidebar 中文件`的方式使文件固定在 Editor 区域，`编辑一个文件`也会将其 Editor 固定。

<h4>Videos</h4>

- [VS Code Top-Ten Pro Tips][u21W_tfPVrY]
- [My Favorite VS Code Extensions][rH1RTwaAeGc]
- [VS Code Can Do That?! VS Code Tips and Tricks][x5GzCohd4eo]
- [Configuring Visual Studio Code For C++ Projects][3Tc6f3nhCxo]
- [CppCon 2017: Rong Lu “C++ Development with Visual Studio Code”][rFdJ68WbkdQ]
- [C++Now 2018: Rong Lu “C++ Development with Visual Studio Code”][erXR6k9TeE]
- [CppCon 2018: Rong Lu “What's new in Visual Studio Code for C++ development”][JME1i3vCRR8]

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [VS Code Top-Ten Pro Tips][u21W_tfPVrY]<br>
2 [SSH with VSCode without internet][ssh-with-vscode-without-internet]<br>
3 [VS Code Tips and Tricks][vscode-tips-and-tricks]<br>
4 [VS Code can do that?!][vscodecandothat]<br>
5 [A curated list of delightful VS Code packages and resources.][awesome-vscode]<br>
6 [Code Format][code-format]<br>
</span>

[u21W_tfPVrY]: https://www.youtube.com/watch?v=u21W_tfPVrY
[marketplace]: https://marketplace.visualstudio.com/VSCode
[gitlens]: https://marketplace.visualstudio.com/items?itemName=eamodio.gitlens
[vscode-icons]: https://marketplace.visualstudio.com/items?itemName=vscode-icons-team.vscode-icons
[vscode-great-icons]: https://marketplace.visualstudio.com/items?itemName=emmanuelbeziat.vscode-great-icons
[ccpp]: https://marketplace.visualstudio.com/items?itemName=ms-vscode.cpptools
[scrollkey]: https://marketplace.visualstudio.com/items?itemName=74th.scrollkey
[remote-ssh]: https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.remote-ssh
[ssh-with-vscode-without-internet]: https://stackoverflow.com/questions/56718453/ssh-with-vscode-without-internet
[bookmarks]: https://github.com/alefragnani/vscode-bookmarks
[bracket-pair-colorizer]: https://github.com/CoenraadS/Bracket-Pair-Colorizer-2
[rH1RTwaAeGc]: https://www.youtube.com/watch?v=rH1RTwaAeGc
[x5GzCohd4eo]: https://www.youtube.com/watch?v=x5GzCohd4eo
[live-server]: https://github.com/ritwickdey/vscode-live-server
[indent-rainbow]: https://github.com/oderwat/vscode-indent-rainbow
[rest-client]: https://github.com/Huachao/vscode-restclient
[vscode-tips-and-tricks]: https://github.com/microsoft/vscode-tips-and-tricks
[3Tc6f3nhCxo]: https://www.youtube.com/watch?v=3Tc6f3nhCxo
[rFdJ68WbkdQ]: https://www.youtube.com/watch?v=rFdJ68WbkdQ
[cmake-tool]: https://github.com/microsoft/vscode-cmake-tools
[erXR6k9TeE]: https://www.youtube.com/watch?v=-erXR6k9TeE
[JME1i3vCRR8]: https://www.youtube.com/watch?v=JME1i3vCRR8
[vscodecandothat]: https://vscodecandothat.com/
[awesome-vscode]: https://github.com/viatsko/awesome-vscode
[shellcheck]: https://github.com/timonwong/vscode-shellcheck
[ClangFormatStyleOptions]: https://clang.llvm.org/docs/ClangFormatStyleOptions.html
[code-format]: https://code.visualstudio.com/docs/cpp/cpp-ide#_code-formatting
[copyright-inserter]: https://github.com/minherz/copyright-inserter
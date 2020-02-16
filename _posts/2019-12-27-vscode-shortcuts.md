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

**C/C++**

- [C/C++][ccpp]

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
    "files.autoGuessEncoding": true
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
]
```

**Tips**

- 如果无法直接在vscode安装插件，可以获取相应的 vsix 文件，并使用 `Install from VSIX` 的方式安装插接件。

- 打开一个新文件会被另一个新打开的文件替换，是因为前一个文件是一个不固定的 Editor（可以通过 Editor 上的字体来判断，斜体表示没有固定，正体表示固定），可以使用`双击 Editor `或者`双击 sidebar 中文件`的方式使文件固定在 Editor 区域，`编辑一个文件`也会将其 Editor 固定。

<h4>Videos</h4>

- [VS Code Top-Ten Pro Tips][u21W_tfPVrY]

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [VS Code Top-Ten Pro Tips][u21W_tfPVrY]<br>
2 [SSH with VSCode without internet][ssh-with-vscode-without-internet]<br>
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
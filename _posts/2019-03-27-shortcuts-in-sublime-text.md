---
layout: post
title: "Sublime Text 中的常用快捷键"
date: 2019-03-27 22:00:00 +0800
tags:
- shortcut
---

在华为工作了近一年的时间，由于大多数时间在写 C，用到最多的编辑器就是 Source Insight，虽然函数调用关系的展示真的牛逼，但多少还是感觉太重了，还是 ST 好用，适合作为各种语言的编辑器。本文列举一些 ST 常用的快捷键。

*本文仅列举 Mac 上的快捷键*

```
command + n                     打开新的标签页
command + w                     关闭当前标签页
command + k + b                 打开或关闭侧栏
control + tab                   切换到上一个标签页（按完之后如果不松开 control 继续按 tab，则会在该方向上继续切换标签页）
control + shift + tab           作用于 control + tab 不松开 control 继续按 tab 键效果相同
command + 1                     跳转到第一个 tab 页，以此类推
command + shift + p             打开命令框
command + ,                     打开 Preference 设置
command + option + [1-4]        分 1 - 4 列
control + [1-4]                 跳到第 1 - 4 列（Group）
control + 0                     Reveal in Side Bar
```

**搜索类**

```
command + f                     搜索
command + shift + f             在文件夹中搜索，可以定多个文件夹
command + g                     搜索下一个
enter                           搜索下一个
command + shift + g             搜索上一个
shift + enter                   搜索上一个
command + p                     搜索文件
```

**编辑类**

```
command + /                     注释当前行或选中行
command + x                     删除当前行
command + d                     选中当前单词，连续按可增加选择下一个相同的单词
command + l                     选中当前行，连续按可增加选择下一行
shift + ↑                       选中当前位置至上一行相同位置，连续按可继续选择
shift + ↓                       选中当前位置至下一行相同位置，连续按可继续选择
command + enter                 在行后插入
command + shift + enter         在行前插入
tab                             向右缩进
shift + tab                     向左缩进
command + z                     撤销
command + y                     恢复撤销
```

**移动类**

```
control + g                     跳转到哪一行
command + r                     跳转到当前文件的某个符号
command + shift + r             跳转到工程的某个符号
command + ->                    跳转到行末
command + <-                    跳转到行首
command + ↓                     跳转到文件末
command + ↑                     跳转到文件首
f12                             跳转到函数定义（有 touch bar 的可以换个key）
shift + f12                     跳转到函数调用处
```

**其他**

```
control + option + b            显示（或关闭）当前行的 git commit 信息，须安装 git blame 插件
```

<h4>My Preferences:</h4>

**Settings**
```
{
    "draw_white_space": "all",
    "font_size": 14,
    "highlight_line": true,
    "ignored_packages":
    [
        "Vintage"
    ],
    "line_padding_bottom": 1,
    "line_padding_top": 1,
    "rulers":
    [
        80,
        120
    ],
    "word_wrap": "auto",
    "save_on_focus_lost": true,
    "translate_tabs_to_spaces": true
}
```

**Key Bindings**
```
[
    { "keys": ["super+]"], "command": "goto_definition" },
    { "keys": ["super+t"], "command": "jump_back" },
    { "keys": ["f7"], "command": "goto_symbol_in_project" },
    { "keys": ["super+up"], "command": "scroll_lines", "args": {"amount": 10.0} },
    { "keys": ["super+down"], "command": "scroll_lines", "args": {"amount": -10.0} },
]
```

<h4>Plugins</h4>

- **[A File Icon](https://packagecontrol.io/packages/A%20File%20Icon)**
- **[ConvertToUTF8](https://github.com/seanliang/ConvertToUTF8)**
- **[Git](https://github.com/kemayo/sublime-text-git)**
- **[Git Blame](https://packagecontrol.io/packages/Git%20blame)**

先列这么多吧，后面再补充 😋

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Sublime Text 3 cheat sheet](/assets/pdf/sublime-text-3-osx.pdf)<br>
2 [Keyboard Shortcuts - OSX](http://docs.sublimetext.info/en/latest/reference/keyboard_shortcuts_osx.html)<br>
3 [Keyboard Shortcuts - Windows/Linux](http://docs.sublimetext.info/en/latest/reference/keyboard_shortcuts_win.html)<br>
</span>

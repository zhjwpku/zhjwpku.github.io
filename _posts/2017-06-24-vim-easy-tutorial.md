---
layout: post
title: Vim Easy Tutorial
date: 2017-06-24 20:00:00 +0800
tags:
- vim
---

记录 Vim 使用技巧 🔨

✈️ :w !sudo tee % ✈️

很多人在编辑需要 root 权限的文件时，如果忘记 sudo，都会使用如上的命令来避免重新编辑文件。但它是什么意思呢？

:help :write
```
:[range]w[rite] [++opt] !{cmd}
                         Execute {cmd} with [range] lines as standard input
                         (note the space in front of the '!').  {cmd} is
                         executed like with ":!{cmd}", any '!' is replaced with
                         the previous command :!.
```

% 是一个寄存器，存储当前文件名。

tee: `ps -ax | tee processes.txt | grep 'foo'` 的图示：

```
+-----------+    tee     +------------+
|           |  --------  |            |
| ps -ax    |  --------  | grep 'foo' |
|           |     ||     |            |
+-----------+     ||     +------------+
                  ||
          +---------------+
          |               |
          | processes.txt |
          |               |
          +---------------+
```

**Make life easier**

在 .vimrc 中添加

```
" Allow saving of files as sudo when I forgot to start vim using sudo.
cmap w!! w !sudo tee > /dev/null %
```

<h4>定位</h4>

```
vim +n file.txt       打开 file.txt 文件，并将光标置于第n行
vim + file.txt        打开 file.txt 文件，并将光标置于最后一行

# 在文件中移动
h(左) j(下) k(上) l(右)
w 下一个词首 e 词尾（如果已是词尾，则定位到下一个词尾）
b 上一个词首
使用数字参数来加速移动，如 10j 表示执行 10 次 j，即向下移动10行

gg        文件首行
shift + g 文件末行

shift + h 本页首行
shift + m 本页中间行
shift + l 本页末行

0 行首(不同环境可能为行首文字)
$ 行末
^ 行首文字

{ 光标移至段落前
} 光标移至段落后

f + <char> 光标定位到匹配的下一个字符
ga 在状态栏显示当前字符的 ASCII 码的十进制、16进制码（Hex）及八进制码（Octal）

mx 标记当前行
'x 将光标置于标记行行首文字

Ctrl + y 上卷一行
Ctrl + e 下卷一行

:n 定位到第n行
```

<h4>编辑</h4>

```
# 删除
dd 删除当前行
de 删除当前字符到下一个词尾的内容（包含词尾）
dw 删除当前字符到下一个词首的内容（不包含词首）
cw 同dw，并转为编辑模式
d0 删至行首
d$ 或 shift+d 删除当前字符到行末的内容

dG      删除当前行在内到末行的内容
d1G/dgg 删除当前行在内到首行的内容

x 删除当前字符

# 复制
yy       复制当前行
5yy      复制当前行在内一下的5行
yG       复制当前行到最后一行的内容
y1G/ygg  复制当前行到第一行的内容
y0       复制到行首(不包含当前字符)
y$       复制当前字符至行尾

# 粘贴
p 在光标所在字符之后粘贴
P 在光标所在字符之前粘贴

:r  file.txt  将 file.txt 文件内容插入到光标所在行之后
:3r file.txt  将 file.txt 文件内容插入到第三行后
:r!ls         将 ls 命令输出内容插入到光标所在行之后

:1,6co9  将1~6行的内容复制到第9行后
:1,6m9   将1~6行的内容移动到第9行后
:1,6d    删除1~6行内容

shift + j 将下一行和当前行合并成一行

:w >> file.txt       将当前文件的所有页面写到file.txt文件最后
:1,12w >> file.txt   将1~12行写到file.txt文件最后
:1,12w newfile.txt   将1~12行写到一个新文件newfile.txt中

</>  缩进/反缩进
gg=G 重新缩进整个文件

u 撤销
shift + U 恢复当前行原始状态
Ctrl +r   重做

# 排序
:%!sort -u  按行重新排序文件内容，重复内容只保留一次（如空行）
:.,+5!sort  排序当前行在内的以下5行内容

:e!                重新载入当前文件，恢复到未修改之前的状态
:e directory       列出directory目录结构，并选择想要打开的文件
:e dir/file.txt    打开文件 file.txt。
:find dir/file.txt 同上
:Bclose            关闭当前显示文件
:Sex/:Explore      打开一个新窗口列出目录结构
```

<h4>窗口操作</h4>

```
:ls   列出打开的文件
:b{n} 显示第{n}个打开的文件
:bn   显示下一个打开的文件
:bp   显示上一个打开的文件
:b#   显示上一个显示的文件，用于最近显示文件的切换
:ba   显示所有打开的文件，多个窗口

:split  打开一个横向的新窗口
:vsplit 打开一个纵向的新窗口
:new    打开一个横向空白窗口
:vnew   打开一个纵向空白窗口
# 注：关闭窗口并不会关闭打开的文件，关闭文件需要使用 :Bclose

# 窗口操作
先按下<Ctrl + w>, 放开后按：
上/下/左/右 或 h/j/k/l 可将光标切换到其他窗口
t (top)跳到最顶上的窗口
b (bottom)跳到最低端的窗口
r (rotate)翻转窗口
= 将窗口设置为等宽的

:qall 退出所有窗口
:res/:resize -8     窗口高度减8行
:res/:resize +8     窗口高度增8行
:vert res/resize -8 窗口宽度减8行
:vert res/resize +8 窗口宽度增8行
```

<h4>文件路径</h4>

```
Ctrl + g  显示当前编辑文件名
:pwd 显示当前文件所在路径
```

<h4>寄存器</h4>

寄存器用于存放 VIM 的临时数据，类似于传统编辑器的剪切板，但 VIM 具有多个寄存器，分别保存不同的临时数据。

```
:help registers 进入寄存器帮助手册

:reg  查看所有寄存器值
:reg  "{register_name}

"ayy  将当前行的内容复制到名为a的寄存器
"ap   将a寄存器中的内容粘贴到光标之后
```
*寄存器用的不多，用的时候使用帮助查看用法*

<h4>map</h4>

用于设置快捷键

```
:help map 查看map帮助文档
:map! xx XXXXX 输入xx时会自动替换为XXXXX
```

<h4>单词补全</h4>

查找当前文档中与键入单词前缀匹配的单词并作提示。

```
Ctrl + n 下一个匹配项
Ctrl + p 上一个匹配项

Ctrl + x + l 查找匹配的行进行补全
Ctrl + x + i 查找当前文件及所包含的文件
Ctrl + x + ] 查找标签中的内容进行匹配
```

<h4>搜索/替换</h4>

```
# 搜索
/ 向下搜索
? 向上搜索
n         搜索下一个
shift + n 搜索上一个

/fast\|path      搜索fast或path
/\<gem\>         搜索gem单词，gemstore不会匹配，"\<" 匹配单词起点，"\>" 匹配单词终点
/\<\d\d\d\d\>    搜索四位数字， "\d" 匹配 0-9 间任一数字，"\D" 匹配非数字
/\<\d\{4}\>      同上，"\{4}" 表示重复4次
/\<[1-9]\d\d\d\> 搜索1000-9999之间的数字
/\<\d\{4}\>      

# 搜索/替换
:s/search_word/replace_word                         将当前行匹配的第一个search_word替换为replace_word
:s/search_word/replace_word/g                       将当前行匹配的所有search_word都替换为replace_word
:<line_beg>,<line_end>s/search_word/replace_word/g  将<line_beg>-<line_end>行间的search_word替换为replace_word
:%s/search_word/replace_word/g                      全文替换

q/ 显示搜索历史
```

<h4>标签页</h4>

```
:tabnew      新建一个标签
:tabnext     转到下一个标签
:tabprevious 转到上一个标签
```

<h4>fold & unfold</h4>

```
:help fold
zo              Open one fold under the cursor.  When a count is given, that
                many folds deep will be opened.  In Visual mode one level of
                folds is opened for all lines in the selected area.

zc              Close one fold under the cursor.  When a count is given, that
                many folds deep are closed.  In Visual mode one level of folds
                is closed for all lines in the selected area.
                'foldenable' will be set.

zM              Close all folds: set 'foldlevel' to 0.
                'foldenable' will be set.

zR              Open all folds.  This sets 'foldlevel' to highest fold level.
```

<h4>不常用命令</h4>

```
.    执行上一次执行的命令，比如插入字符或删除一行，在命令行执行 . 会重复刚才的操作
:!ls 查看目录文件列表，！可以从Vim临时切换到Shell，使用 ！可以方便查看查看
gUU  行大写
guu  行小写
g~~  翻转行内字符大小写
guw  当前字符到词尾小写
gUw  当前字符到词尾大写
g~w  当前字符到词尾进行翻转
veU  同gUw
veu  同guw
ve~  同g~w
vEU  当前字符到下一个空白符前大写
vEu  当前字符到下一个空白符前小写
vE~  当前字符到下一个空白符前翻转

ggguG 全文小写
gggUG 全文大写
ggg~G 全文翻转

## 命令历史
:history 列出命令历史记录 (包含q/ & q:)
q/       搜索命令历史
q:       命令行历史

## 编译选项
F9   编译或执行当前程序
:cn  定位下一个出错点，不管是否在当前文件中
:cc  显示当前错误的输出信息
:cl  显示所有错误列表

:%!xxd    以十六进制查看
:%!xxd -r 从十六进制返回
```

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [How does the vim “write with sudo” trick work?](https://stackoverflow.com/questions/2600783/how-does-the-vim-write-with-sudo-trick-work)<br>
</span>

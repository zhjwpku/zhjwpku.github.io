---
layout: post
title: Python 名称校验
date: 2018-05-09 22:00:00 +0800
tags:
- regex
- utf8
---

最近的一个需求，对前端传来的变量名（unicode字符串）进行校验，变量名只能包含汉字、数字、字母和下划线，且长度不能超过50（每个汉字算两个字符）。下面是笔者的一个实现。

*test.py*

```
import re

def is_name_validate(name):
    length = len(name)
    print(length)

    utf8_length = len(name.encode('utf8'))
    print(name.encode('utf8'))
    print(utf8_length)

    real_length = (utf8_length - length) / 2 + length

    if real_length > 50:
        return False

    pattern = re.compile(u'^[\u4e00-\u9fa5A-Za-z0-9_]*$')
    if pattern.match(name):
        return True
    else:
        return False

if __name__ == '__main__':
    print(is_name_validate(u'00000000001111111111222222222233333333334444444你好'))
```

执行结果：

```
➜  ~ python test.py
49
b'00000000001111111111222222222233333333334444444\xe4\xbd\xa0\xe5\xa5\xbd'
53
False
```

---
layout: post
title: JavaScript 常用包功能介绍
date: 2017-04-28 01:00:00 +0800
tags:
- javascript
---

JavaScript 有太多可用的包了，让笔者这样的初学者惊慌失措，好记心不如烂笔头，本文将不间断更新笔者学习 JavaScript 过程中遇到的包。

<h4><a href="https://github.com/zertosh/invariant/blob/master/invariant.js">invariant</a></h4>

用法:

```javascript
import invariant from 'invariant'

invariant(condition, message_format, param1)
```

如果 condition 为真值，则不打印错误，否则打印 message_format, 字符串可以带最多6个参数。类似的包 [warning][warning]。

其源码非常简单：

```javascript
'use strict';

/**
 * Use invariant() to assert state which your program assumes to be true.
 *
 * Provide sprintf-style format (only %s is supported) and arguments
 * to provide information about what broke and what you were
 * expecting.
 *
 * The invariant message will be stripped in production, but the invariant
 * will remain to ensure logic does not differ in production.
 */

var NODE_ENV = process.env.NODE_ENV;

// 最多6个参数值，超出的打印为 undefined，这样的实现有点丑...
var invariant = function(condition, format, a, b, c, d, e, f) {
  if (NODE_ENV !== 'production') {
    if (format === undefined) {
      throw new Error('invariant requires an error message argument');
    }
  }

  if (!condition) {
    var error;
    if (format === undefined) {
      error = new Error(
        'Minified exception occurred; use the non-minified dev environment ' +
        'for the full error message and additional helpful warnings.'
      );
    } else {
      var args = [a, b, c, d, e, f];
      var argIndex = 0;
      error = new Error(
        format.replace(/%s/g, function() { return args[argIndex++]; })
      );
      error.name = 'Invariant Violation';
    }

    error.framesToPop = 1; // we don't care about invariant's own frame
    throw error;
  }
};

module.exports = invariant;
```

<h4><a href="https://github.com/nuysoft/Mock">Mock</a></h4>

生成随机数据，拦截 Ajax 请求。用于前端独立于后端进行开发。使用手册：[https://github.com/nuysoft/Mock/wiki/Getting-Started][mock]。

<h4><a href="https://github.com/reactjs/prop-types">prop-types</a></h4>

用于在运行时检查传递给 Components 的 props 的类型是否正确。

[invariant]: https://github.com/zertosh/invariant
[warning]: https://github.com/BerkeleyTrue/warning
[mock]: https://github.com/nuysoft/Mock/wiki/Getting-Started

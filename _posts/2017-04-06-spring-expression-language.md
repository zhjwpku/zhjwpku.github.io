---
layout: post
title: "Spring Expression Language (SpEL)"
date: 2017-04-06 12:00:00 +0800
tags:
- spel
---

[SpEL][spel] 是一个支持运行时查询和操作对象图的强大的表达式语言，其语法类似于 Unified EL 并且提供了诸如方法调用和基本字符串模板功能等附加特性。

SpEL 支持以下功能：

    - Literal expressions                         文字表达式
    - Boolean and relational operators            布尔表达式及关系操作符
    - Regular expressions                         正则表达式
    - Class expressions                           类表达式
    - Accessing properties, arrays, lists, maps   获取系统属性，array，lists，maps
    - Method invocation                           方法调用
    - Relational operators                        关系表达式
    - Assignment                                  赋值
    - Calling constructors                        调用构造函数
    - Bean references                             参考Bean
    - Array construction                          数组构造
    - Inline lists                                内嵌链表
    - Inline maps                                 内嵌maps
    - Ternary operator                            三目表达式
    - Variables                                   变量
    - User defined functions                      用户自定义函数
    - Collection projection                       集合映射
    - Collection selection                        集合选择
    - Templated expressions                       模板表达式

看完以下两个教程能够对 SpEL 有个明白的理解：

- [官方文档][spel]
- [Spring表达式语言 之 5.3 SpEL语法 —— 跟我学spring3][ref1]

本文增加笔者实际应用中的一个需求，根据一组 K/V 值及一个条件语言 condition，使用 SpEL 计算这组 K/V 值是否满足 condition。

代码如下：

{% highlight java %}
public ResponseEntity<Version> latestNameVersionPost(
    @PathVariable("name") String name,
    @PathVariable("version") Integer version,
    @RequestBody List<Property> clientInfo) {

    Map<String, String> map = new HashMap<String, String>();
    for (Property p: clientInfo) {
        map.put(p.getK(), p.getV());
    }
    map.put("name", name);
    map.put("version", version.toString());

    ExpressionParser parser = new SpelExpressionParser();
    
    // 设置context
    EvaluationContext context = new StandardEvaluationContext();
    context.setVariable("properties", map);

    // 条件，实际需求中需要从数据库中获取不同的condition进行处理
    String condition = "#properties['version'] > '3.1' and #properties['hello'] == 'world' or #properties['patch'] == 'true'";
    
    // 计算是否满足
    boolean satisfy = parser.parseExpression(condition).getValue(context, boolean.class);

    System.out.println(satisfy);

    // do some magic!
    return new ResponseEntity<Version>(HttpStatus.OK);
}
{% endhighlight %}

[spel]: https://docs.spring.io/spring/docs/current/spring-framework-reference/html/expressions.html
[ref1]: http://sishuok.com/forum/blogPost/list/2463.html

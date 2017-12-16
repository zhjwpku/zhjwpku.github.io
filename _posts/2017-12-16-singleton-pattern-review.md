---
layout: post
title: 再看单例模式
date: 2017-12-16 15:00:00 +0800
tags:
- DesignPattern
---

当看完《Head First Design Patterns》一书之后，你不一定记得住所有的设计模式，你需要在阅读或编写代码的过程中循序渐进地掌握每一种设计模式，做到所谓的各个击破。本文笔者就先来把Singleton这颗蛋吃掉。

徒手写一个线程不安全的Singleton:

```java
public class Singleton {
    private static Singleton instance = null;

    private Singleton() {}  //  私有化构造函数，使其不能外部实例化

    public static getInstance() {   // 好吧，说实话，徒手写忘了加static
        if (instance == null) {
            instance = new Singleton();
        }

        return instance;
    }
}
```

上面的 `getInstance` 可以被多个线程访问，因此不难理解各个线程可能获得不同的对象，违背了单例的原则。

然后为了避免上面的情况，我们可以使用 `synchronized` 来迫使每个线程在进入这个方法之前先等候其他线程离开，于是有了单例的第二个实现:

```java
public class Singleton {
    private static Singleton instance = null;

    private Singleton() {}  //  私有化构造函数，使其不能外部实例化

    public static synchronized getInstance() {
        if (instance == null) {
            instance = new Singleton();
        }

        return instance;
    }
}
```

但是这个代码又有问题了，每次获取实例都需要等待其它线程的离开，极大影响性能。单例初始化完成之后，之后的同步等待就没有必要了。因此我们使用**双重检查加锁**的方式来写第三个版本的单例模式:

```java
public class Singleton {
    private static Singleton instance = null;

    private Singleton() {}  //  私有化构造函数，使其不能外部实例化

    public static getInstance() {   // 不再同步整个函数
        if (instance == null) {
            synchronized (Singleton.class) {
                instance = new Singleton();
            }
        }

        return instance;
    }
}
```

但是由于编译器的**指令重排**，上述代码依然不是绝对线程安全的。指令重排什么意思呢？通俗的讲，Java编译器会将 instance = new Singleton() 指令编译成如下JVM指令:

```
memory = allocate();    // 分配内存
ctorInstance(memory);   // 初始化对象
instance = memory;      // 设置内存指向
```

但是由于指令重排，上述指令的顺序可能变成了:

```
memory = allocate();    // 分配内存
instance = memory;      // 设置内存指向
ctorInstance(memory);   // 初始化对象
```

这样当在第二个线程判断 if (instance == null) 返回 false 的时候，上述的第三个指令可能还未被第一个线程执行，因此第二个线程获取的 instance 对象是未初始化的。

解决这个问题需要使用 volatile 指令，于是有了最终版的线程安全的单例模式:

```java
public class Singleton {
    private volatile static Singleton instance = null;  // 禁止指令重排

    private Singleton() {}  //  私有化构造函数，使其不能外部实例化

    public static getInstance() {   // 不再同步整个函数
        if (instance == null) {
            synchronized (Singleton.class) {
                instance = new Singleton();
            }
        }

        return instance;
    }
}
```

以上写法属于**懒汉模式**，还有一种**饿汉模式**，即单例对象一开始就被 new Singleton() 主动构建，利用这个做法，我们依赖 JVM 在加载这个类时马上创建唯一的单例实例。JVM 保证任何线程访问 instance 静态变量之前，一定先创建此实例。

```java
public class Singleton {
    private static Singleton intance = new Singleton();

    private Singleton() {}

    public static Singleton getInstance() {
        return intance;
    }
}
```

现实中我们会遇到很多饿汉模式的单例实现，如 `Runtime` 类:

```java
public class Runtime {
    private static Runtime currentRuntime = new Runtime();

    public static Runtime getRuntime() {
        return currentRuntime;
    }

    private Runtime() {}

    // below are other useful methods
}
```

ok，以上就是我们面对大多数面试官时你需要写出单例模式。但是单例模式存在一个问题: 无法防止通过反射来重复构建对象:

```java
// 获得构造器
Constructor con = Singleton.class.getDeclaredConstructor();
// 设置可访问
con.setAccessible(true);
// 构造两个不同的对象
Singleton singleton1 = (Singleton)con.newInstance();
Singleton singleton2 = (Singleton)con.newInstance();
```

那么如何实现一个可以防止反射的单例模式呢？可以使用枚举实现单例，这是一种优雅而又简洁的方式:

```java
public enum SingletonEnum {
    INSTANCE;
}
```

枚举的方式笔者了解的还不透彻，等掌握了再来补充。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 Freeman, Eric,, Freeman, Elisabeth., Sierra, Kathy.Bates, Bert., eds. Head First Design Patterns. Sebastopol, CA : O’Reilly, 2004. Print.<br>
2 微信公众号: TheAlgorithm. 漫画: 什么是单例模式? (整合版)
</span>

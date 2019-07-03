---
layout: post
title: "Lamport Timestamps"
date: 2019-07-01 00:00:00 +0800
categories: paper
tags:
- paper
---

[Lamport timestamps][lamporttimestamps] 是一种在分布式系统中获取全序关系(total ordering)的简单算法。

Paper: [Lamport, L. (1978). Time, clocks, and the ordering of events in a distributed system][paper]

**The Partial Ordering**

论文定义了 "happened before" 关系，用`→`表示，作用于系统中的事件，并满足如下条件:

```
1) If `a` and `b` are events in the same process, and `a` comes before `b`, then `a` → `b`
2) If `a` is the sending of a message by one process and `b` is the receipt of the same message by another process, then `a` → `b`
3) If `a` → `b` and `b` → `c` then `a` → `c`
```

当两个独立事件满足 `a ↛ b` 且 `b ↛ a` 则认为 `a` 和 `b` 是并发(cuncurrent)的。

**Logical Clocks**

定义逻辑时钟 Ci，其中 i 为进程标识，`Ci<a>` 代表进程 i 上事件 a 的发生时间。

```
Clock Condition. For any events `a`, `b`:
                            if `a` → `b` then C<a> < C<b>

C1. If `a` and `b` are events in process Pi, and `a` comes before `b`, then Ci<a> < Ci<b>
C2. If `a` is the sending of a message by Pi, and `b` is the receipt of that message by Pj, then Ci<a> < Cj<b>
```

注意该关系反过来并不成立，例如论文图中的 p2 和 q3。

为了满足上面的 Clock Condition，论文提出了逻辑时钟的实现规则(Implementation Rule):

```
IR1. Each process Pi increments Ci between any two successive events.
IR2. a) If `a` is the sending of a message m by P1, then the message m contains a timestamp Tm = Ci<a>
IR2. b) Upon receiving a message m, Pj sets Cj greater than or equal to its present value and greater than Tm
```

上面的算法可以用如下伪代码表示：

```
Sending process
# event is known
time = time + 1;
# event happens
send(message, time);

Receiving Process
(message, time_stamp) = receive();
time = max(time_stamp, time) + 1;
```

**Ordering the Events Totally**

IR1 和 IR2 保证了系统逻辑时钟的正确性，据此可以实现系统事件的全序关系，用 ⇒ 表示。

对于 Pi 和 Pj，使用 ≺ 来表示他们之间的一种随意关系，因此可以使用如下算法来获得全序关系:

```
If `a` is an event in Pi and `b` is an event in Pj, then `a` ⇒ `b` iff
1) Ci<a> < Cj<b>
or
2) Ci<a> = Cj<b> and Pi ≺ Pj
```

论文之后根据全序关系实现了一种多进程获取同一资源的分布式算法，算法定义了 5 条规则，这里不赘述。注意，该算法的一个前提是不能有失效节点，否则会一直等待失效节点的消息到来。

**Anomalous Behavior**

上面的算法存在异常行为。当存在系统外部事件的时候，系统是感知不到的，因此假如事件 A 先于事件 B，对于 ⇒ 关系，系统也只能认为 A 和 B 是并发的。

**Physical Clocks**

论文提出了使用物理时钟来消除上述异常行为的方法。通过给出物理时钟的约束，来保证事件在此基础上的全序关系。

虽然给出了一系列推理，但我认为这不是该 Paper 的重点，在此也不过多描述。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Time and Ordering: Lamport Timestamps](https://www.youtube.com/watch?v=CMBjvCzDVkY)<br>
</span>

[lamporttimestamps]: https://en.wikipedia.org/wiki/Lamport_timestamps
[paper]: https://lamport.azurewebsites.net/pubs/time-clocks.pdf

---
layout: post
title: 半小时总结一下阿里中间件比赛
date: 2019-07-22 00:00:00 +0800
tags:
- dubbo
- tianchi
---

机缘巧合之下了解到阿里天池有个 [POLARDB数据库性能大赛](https://tianchi.aliyun.com/competition/entrance/231689/introduction)，但是今年的赛程还没开始，在官网上又看到了正在进程中[中间件性能挑战赛](https://tianchi.aliyun.com/competition/entrance/231714/introduction)，于是报名参加，了解一下比赛的流程，为后面的数据库性能大赛趟趟路。

初赛的题目是写一个 dubbo 的负载均衡算法，dubbo 本身提供了 RandomLoadBalance/RoundRobinLoadBalance/LeastActiveLoadBalance/ConsistentHashLoadBalance 四种 LB 算法，因此这个题目不算难，但是想要拿到好的成绩也不简单。

由于这次参赛只是为了了解流程，笔者没有尝试过多的算法，只是根据不同 server 的线程数设计了一个带权重的随机算法，附加根据错误调用的反馈动态调整权重，代码如下：

```java
// repo address: git@code.aliyun.com:zhjwpku/adaptive-loadbalance.git
// 负载均衡的类
public class UserLoadBalance implements LoadBalance {
    Timer timer;

    private static volatile int smallPoolSize = 0;
    private static volatile int mediumPoolSize = 0;
    private static volatile int largePoolSize = 0;
    private static volatile ConcurrentHashMap<String, Integer> map = null;

    public UserLoadBalance() {
        timer = new Timer();
        timer.schedule(new TimerTask() {
            @Override
            public void run() {
                restPoolSize();
            }
        }, 0, 2000);
    }

    private static ConcurrentHashMap<String, Integer> getMap(){
        if (null == map){
            synchronized(ConcurrentHashMap.class){
                if (null == map){
                    map = new ConcurrentHashMap<>();
                    map.put("provider-large", largePoolSize);
                    map.put("provider-medium", mediumPoolSize);
                    map.put("provider-small", smallPoolSize);
                }
            }
        }
        return map;
    }

    public static void setPoolSize(String host, int poolSize) {
        switch (host) {
            case "provider-small":
                smallPoolSize = poolSize;
                break;
            case "provider-medium":
                mediumPoolSize = poolSize + 100;
                break;
            case "provider-large":
                largePoolSize = poolSize + 150;
                break;
            default:
                System.out.println("Error occurred");
        }
    }

    private static void restPoolSize() {
        ConcurrentHashMap<String, Integer> map = getMap();
        synchronized (map) {
            int tmpSSize = map.get("provider-small");
            int tmpMSize = map.get("provider-medium");
            int tmpLSize = map.get("provider-large");
            //System.out.println("s: " + tmpSSize + "; m: " + tmpMSize + "; l: " + tmpLSize);
            map.put("provider-small", (smallPoolSize + tmpSSize) / 2);
            map.put("provider-medium", (mediumPoolSize + tmpMSize) / 2);
            map.put("provider-large", (largePoolSize + tmpLSize) / 2);
        }
    }

    public static void decPoolSize(String host) {
        ConcurrentHashMap<String, Integer> map = getMap();
        synchronized (map) {
            Integer preWeight = map.get(host);
            switch (host) {
                case "provider-small":
                    map.put(host, Math.max(preWeight - 10, smallPoolSize - 100));
                    break;
                case "provider-medium":
                    map.put(host, Math.max(preWeight - 10, mediumPoolSize - 200));
                    break;
                case "provider-large":
                    map.put(host, Math.max(preWeight - 10, largePoolSize - 300));
                    break;
                default:
                    System.out.println("Error occurred");
            }
        }
    }

    public static int getPoolSize(String host) {
        return getMap().get(host);
    }

    @Override
    public <T> Invoker<T> select(List<Invoker<T>> invokers, URL url, Invocation invocation) throws RpcException {
        int[] weights = new int[3];

        for (int i = 0; i < 3; i++) {
            Invoker<T> invoker = invokers.get(i);
            String host = invoker.getUrl().getHost();
            int poolSize = getPoolSize(host);
            weights[i] = poolSize;
        }

        int totalWeight = weights[0] + weights[1] + weights[2];

        int offset = ThreadLocalRandom.current().nextInt(totalWeight);

        for (int i = 0; i < 3; i++) {
            offset -= weights[i];
            if (offset < 0)
                return invokers.get(i);
        }

        return invokers.get(ThreadLocalRandom.current().nextInt(invokers.size()));
    }
}

// 反馈的类
@Activate(group = Constants.CONSUMER)
public class TestClientFilter implements Filter {

    @Override
    public Result invoke(Invoker<?> invoker, Invocation invocation) throws RpcException {
        try {
            return invoker.invoke(invocation);
        } catch (Exception e) {
            //isSuccess = false;
            throw e;
        }
    }

    @Override
    public Result onResponse(Result result, Invoker<?> invoker, Invocation invocation) {
        String host = invoker.getUrl().getHost();
        if (result.getValue() != null) {
            //UserLoadBalance.incPoolSize(host);
        } else {
            UserLoadBalance.decPoolSize(host);
        }
        return result;
    }
}

```

最终的成绩是 `219 /1259581/1259581/27532`，对比一下第一名 `1318278/1318278/28127` 和晋级的最后一名 `1263413/1263413/28025`。

至于如何调优，看到有人写用 CompletableFuture，但毕竟笔者不写 Java 很久了，也没有花更多的心思在上面，也就这样了。

虽然没有进复赛，但是复赛的题目挺有意思 —— 《实现一个进程内基于队列的消息持久化存储引擎》，看样子是要实现一个 K/V 存储，不过语言限制为 JAVA，笔者也不去看题目细节了，进入复赛的小伙伴们加油！

P.S. 参加比赛了解到一个 HTTP benchmarking 的工具 [wrk](https://github.com/wg/wrk)，它包含了 Redis、Nginx、LuaJIT 的一些组件，后面笔者会仔细学习一下。


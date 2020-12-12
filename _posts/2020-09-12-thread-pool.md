---
layout: post
title: Thread Pool
date: 2020-09-12 12:00:00 +0800
tags:
- C/C++
---

本文使用 C++ 实现了一个简单的线程池。

```c++
#pragma once

#include <thread>
#include <mutex>
#include <condition_variable>
#include <queue>
#include <functional>

class ThreadPool {
 public:
    ThreadPool(int thread_cnt = std::thread::hardware_concurrency()) : stop(false) {
      if (thread_cnt <= 0) {
          thread_cnt = std::thread::hardware_concurrency();
      }

      for (auto i = 0; i < thread_cnt; i++) {
        threads.emplace_back(std::thread([this]() {
                      while(true) {
                        std::unique_lock<std::mutex> lck(this->mu);
                        this->cond.wait(lck, [this] { return this->stop || !this->tasks.empty(); });

                        if (this->stop && this->tasks.empty()) {
                          return;
                        }
                        auto func = this->tasks.front();
                        this->tasks.pop();
                        lck.unlock();
                        func();
                      }
                    }));
      }
    }

    ~ThreadPool() {
      std::unique_lock<std::mutex> lck(mu);
      stop = true;
      lck.unlock();
      cond.notify_all();

      for (auto &t: threads) {
        t.join();
      }
    }

    template<typename Func, typename... Args>
    void enqueue(Func&& func, Args&&... args) {
      std::lock_guard<std::mutex> lg(mu);
      tasks.push([=]() { func(args...); });
      cond.notify_one();
    }

    ThreadPool(const ThreadPool& other) = delete;
    ThreadPool& operator=(const ThreadPool &other) = delete;

 private:
    std::vector<std::thread> threads;
    std::queue<std::function<void()>> tasks;
    std::mutex mu;
    std::condition_variable cond;
    bool stop;
};
```

上述实现单纯只是将任务放到队列中，线程在空闲的时候会从队列中取一个任务执行，但是没有办法获取返回值。在 [progschj/ThreadPool](https://github.com/progschj/ThreadPool/blob/master/ThreadPool.h) 的实现中，使用 `future` 和 `packaged_task` 这两个特性，实现了可以获取任务的返回值，极大的提高了代码的通用性。下面贴出其源码:

```c++
#ifndef THREAD_POOL_H
#define THREAD_POOL_H

#include <vector>
#include <queue>
#include <memory>
#include <thread>
#include <mutex>
#include <condition_variable>
#include <future>
#include <functional>
#include <stdexcept>

class ThreadPool {
public:
    ThreadPool(size_t);
    template<class F, class... Args>
    auto enqueue(F&& f, Args&&... args)
        -> std::future<typename std::result_of<F(Args...)>::type>;
    ~ThreadPool();
private:
    // need to keep track of threads so we can join them
    std::vector< std::thread > workers;
    // the task queue
    std::queue< std::function<void()> > tasks;

    // synchronization
    std::mutex queue_mutex;
    std::condition_variable condition;
    bool stop;
};

// the constructor just launches some amount of workers
inline ThreadPool::ThreadPool(size_t threads)
    :   stop(false)
{
    for(size_t i = 0;i<threads;++i)
        workers.emplace_back(
            [this]
            {
                for(;;)
                {
                    std::function<void()> task;

                    {
                        std::unique_lock<std::mutex> lock(this->queue_mutex);
                        this->condition.wait(lock,
                            [this]{ return this->stop || !this->tasks.empty(); });
                        if(this->stop && this->tasks.empty())
                            return;
                        task = std::move(this->tasks.front());
                        this->tasks.pop();
                    }

                    task();
                }
            }
        );
}

// add new work item to the pool
template<class F, class... Args>
auto ThreadPool::enqueue(F&& f, Args&&... args)
    -> std::future<typename std::result_of<F(Args...)>::type>
{
    using return_type = typename std::result_of<F(Args...)>::type;

    auto task = std::make_shared< std::packaged_task<return_type()> >(
            std::bind(std::forward<F>(f), std::forward<Args>(args)...)
        );

    std::future<return_type> res = task->get_future();
    {
        std::unique_lock<std::mutex> lock(queue_mutex);

        // don't allow enqueueing after stopping the pool
        if(stop)
            throw std::runtime_error("enqueue on stopped ThreadPool");

        tasks.emplace([task](){ (*task)(); });
    }
    condition.notify_one();
    return res;
}

// the destructor joins all threads
inline ThreadPool::~ThreadPool()
{
    {
        std::unique_lock<std::mutex> lock(queue_mutex);
        stop = true;
    }
    condition.notify_all();
    for(std::thread &worker: workers)
        worker.join();
}

#endif
```

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Simple thread pool](https://vorbrodt.blog/2019/02/12/simple-thread-pool/).<br>
2 [highwayhash/highwayhash/data_parallel.h](https://github.com/google/highwayhash/blob/9099074416ebc926c9e5e6f5143db92ebd9b4c03/highwayhash/data_parallel.h)<br>
</span>

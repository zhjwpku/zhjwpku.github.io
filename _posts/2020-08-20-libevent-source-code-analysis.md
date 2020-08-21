---
layout: post
title: libevent 源码分析
date: 2020-08-20 20:00:00 +0800
tags:
- libevent
- epoll
---

[Libevent][libevent] 是一个C语言编写的、轻量级的开源高性能事件驱动库，支持 epoll、kqueue、/dev/poll、select、poll 等多种 I/O 多路复用。在学习 [UNIX Network Programming][unp] 的时候笔者曾将书中的 [tcpservpoll01.c][tcpservpoll01] 改写为 epoll 方式的 [echo server][tcpservepoll01]。本文首先对 libevent 做一个简要的源码分析，然后给出两个使用 libevent 实现的 echo server。

<h4>源码分析</h4>

我们首先给出 libevent API 基本的调用顺序，然后对这些接口进行简要分析。

- event_base_new()
- event_new()
- event_add()
- event_base_dispatch()

如上是 libevent 中最基本的四个接口，下面分别进行分析。

**[event_base_new][event_base_new]**

[event_base][event_base] 是 libevent 的核心结构体，它可以承载一系列事件结构，并查看事件的到来。event_base_new 用来创建这个结构，该函数首先调用 [event_config_new][event_config_new] 创建一个 event_config，然后调用 event_base_new_with_config 来创建并初始化 event_base 结构。在一些应用中，可以通过配置 event_config（如指定边沿触发、忽略环境变量等）来定制 event_base 的初始化行为。

```c
struct event_base *
event_base_new_with_config(const struct event_config *cfg)
{
    int i;
    struct event_base *base;

    if ((base = mm_calloc(1, sizeof(struct event_base))) == NULL) {
        event_warn("%s: calloc", __func__);
        return NULL;
    }

    if (cfg)
        base->flags = cfg->flags;

    ...
    // 忽略了一些语句，我们主要看下 event_base 是如何选取底层I/O多路复用操作的

    base->evbase = NULL;

    for (i = 0; eventops[i] && !base->evbase; i++) {
        if (cfg != NULL) {
            /* determine if this backend should be avoided */
            if (event_config_is_avoided_method(cfg,
                eventops[i]->name))
                continue;
            if ((eventops[i]->features & cfg->require_features)
                != cfg->require_features)
                continue;
        }

        base->evsel = eventops[i];

        base->evbase = base->evsel->init(base);
    }

    if (base->evbase == NULL) {
        event_warnx("%s: no event mechanism available",
            __func__);
        base->evsel = NULL;
        event_base_free(base);
        return NULL;
    }

    if (evutil_getenv_("EVENT_SHOW_METHOD"))
        event_msgx("libevent using: %s", base->evsel->name);

    /* allocate a single active event queue */
    if (event_base_priority_init(base, 1) < 0) {
        event_base_free(base);
        return NULL;
    }

    return (base);
}
```

上面一个重要的函数结构是 [eventops][eventops]:

```c
/* Array of backends in order of preference. */
static const struct eventop *eventops[] = {
#ifdef EVENT__HAVE_EVENT_PORTS
    &evportops,
#endif
#ifdef EVENT__HAVE_WORKING_KQUEUE
    &kqops,
#endif
#ifdef EVENT__HAVE_EPOLL
    &epollops,
#endif
#ifdef EVENT__HAVE_DEVPOLL
    &devpollops,
#endif
#ifdef EVENT__HAVE_POLL
    &pollops,
#endif
#ifdef EVENT__HAVE_SELECT
    &selectops,
#endif
#ifdef _WIN32
    &win32ops,
#endif
#ifdef EVENT__HAVE_WEPOLL
    &wepollops,
#endif
    NULL
};
```

libevent 在安装的配置阶段会依据环境生成 event-config.h，上面的宏都定义在这个文件中。初始化函数按照顺序查找最优的 I/O 多路复用方式，将其对应的结构赋值给 base->evsel，调用 `base->evsel->init(base)` 后将生成的结构赋给 `base->evbase`。以 epoll 为例，其 epollops 定义如下:

```c
const struct eventop epollops = {
    "epoll",                    // const char *name;
    epoll_init,                 // void *(*init)(struct event_base *);
    epoll_nochangelist_add,     // int (*add)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
    epoll_nochangelist_del,     // int (*del)(struct event_base *, evutil_socket_t fd, short old, short events, void *fdinfo);
    epoll_dispatch,             // int (*dispatch)(struct event_base *, struct timeval *);
    epoll_dealloc,
    1, /* need reinit */
    EV_FEATURE_ET|EV_FEATURE_O1|EV_FEATURE_EARLY_CLOSE,
    0
};
```

先看 epoll 对应的初始化接口 [epoll_init][epoll_init]:

```c
static void *
epoll_init(struct event_base *base)
{
    epoll_handle epfd = INVALID_EPOLL_HANDLE;
    struct epollop *epollop;

#ifdef EVENT__HAVE_EPOLL_CREATE1
    /* First, try the shiny new epoll_create1 interface, if we have it. */
    epfd = epoll_create1(EPOLL_CLOEXEC);
#endif
    if (epfd == INVALID_EPOLL_HANDLE) {
        /* Initialize the kernel queue using the old interface.  (The
        size field is ignored   since 2.6.8.) */
        if ((epfd = epoll_create(32000)) == INVALID_EPOLL_HANDLE) {
            if (errno != ENOSYS)
                event_warn("epoll_create");
            return (NULL);
        }
#ifndef EVENT__HAVE_WEPOLL
        evutil_make_socket_closeonexec(epfd);
#endif
    }

    if (!(epollop = mm_calloc(1, sizeof(struct epollop)))) {
        close_epoll_handle(epfd);
        return (NULL);
    }

    epollop->epfd = epfd;

    /* Initialize fields */
    epollop->events = mm_calloc(INITIAL_NEVENT, sizeof(struct epoll_event));
    if (epollop->events == NULL) {
        mm_free(epollop);
        close_epoll_handle(epfd);
        return (NULL);
    }
    epollop->nevents = INITIAL_NEVENT;

#ifndef EVENT__HAVE_WEPOLL
    if ((base->flags & EVENT_BASE_FLAG_EPOLL_USE_CHANGELIST) != 0 ||
        ((base->flags & EVENT_BASE_FLAG_IGNORE_ENV) == 0 &&
        evutil_getenv_("EVENT_EPOLL_USE_CHANGELIST") != NULL)) {

        base->evsel = &epollops_changelist;
    }
#endif
    ...
    // 忽略一些语句

    return (epollop);
}
```

epoll_init 调用 `epoll_create1` 或 `epoll_create` 给 epoll 分配一个文件描述符，然后创建初始的事件列表（大小为 `define INITIAL_NEVENT 32`个），并将它们填到 epollop，之后 epollop 作为返回结构被赋值给 event_base 的 evbase。

本文忽略了 event_base_new 的一些其它工作。至此，event_base 的初始化流程分析完毕，下面来看 `event_new` 和 `event_add`。

**[event_new][event_new]**

event_new 分配一个 event 结构，并使用入参（包括 event_base、事件发生的描述符、事件的种类、事件回调及回调参数）对其进行初始化。该函数返回的 event 的状态为 `non-pending`，也就是还没有添加到 I/O 多路复用结构及 event_base 相应的结构。还有一点要说明的是，该函数没有指定超时时间，事件的超时时间是在 event_add 时指定的。

```c
struct event *
event_new(struct event_base *base, evutil_socket_t fd, short events, void (*cb)(evutil_socket_t, short, void *), void *arg)
{
    struct event *ev;
    ev = mm_malloc(sizeof(struct event));
    if (ev == NULL)
        return (NULL);
    if (event_assign(ev, base, fd, events, cb, arg) < 0) {
        mm_free(ev);
        return (NULL);
    }

    return (ev);
}
```

**[event_add][event_add]**

event_add 调用 event_add_nolock_ 将事件添加到相应的 I/O 多路复用结构及 event_base 相应的结构中，该函数成功后，所添加的事件变为 pending 状态。

```c
int
event_add_nolock_(struct event *ev, const struct timeval *tv,
    int tv_is_absolute)
{
    struct event_base *base = ev->ev_base;
    int res = 0;
    int notify = 0;

    ...
    // 忽略一些语句

    if ((ev->ev_events & (EV_READ|EV_WRITE|EV_CLOSED|EV_SIGNAL)) &&
        !(ev->ev_flags & (EVLIST_INSERTED|EVLIST_ACTIVE|EVLIST_ACTIVE_LATER))) {
        if (ev->ev_events & (EV_READ|EV_WRITE|EV_CLOSED))
            res = evmap_io_add_(base, ev->ev_fd, ev);
        else if (ev->ev_events & EV_SIGNAL)
            res = evmap_signal_add_(base, (int)ev->ev_fd, ev);
        if (res != -1)
            event_queue_insert_inserted(base, ev);
        if (res == 1) {
            /* evmap says we need to notify the main thread. */
            notify = 1;
            res = 0;
        }
    }

    ...
    // 忽略一些语句

    return (res);
}
```

本文暂时只关注上面的 `evmap_io_add_`:

```c
/* return -1 on error, 0 on success if nothing changed in the event backend,
 * and 1 on success if something did. */
int
evmap_io_add_(struct event_base *base, evutil_socket_t fd, struct event *ev)
{
    const struct eventop *evsel = base->evsel;
    struct event_io_map *io = &base->io;
    struct evmap_io *ctx = NULL;
    int nread, nwrite, nclose, retval = 0;
    short res = 0, old = 0;
    struct event *old_ev;

    EVUTIL_ASSERT(fd == ev->ev_fd);

    if (fd < 0)
        return 0;

#ifndef EVMAP_USE_HT
    if (fd >= io->nentries) {
        if (evmap_make_space(io, fd, sizeof(struct evmap_io *)) == -1)
            return (-1);
    }
#endif
    GET_IO_SLOT_AND_CTOR(ctx, io, fd, evmap_io, evmap_io_init,
                         evsel->fdinfo_len);

    nread = ctx->nread;
    nwrite = ctx->nwrite;
    nclose = ctx->nclose;

    if (nread)
        old |= EV_READ;
    if (nwrite)
        old |= EV_WRITE;
    if (nclose)
        old |= EV_CLOSED;

    if (ev->ev_events & EV_READ) {
        if (++nread == 1)
            res |= EV_READ;
    }
    if (ev->ev_events & EV_WRITE) {
        if (++nwrite == 1)
            res |= EV_WRITE;
    }
    if (ev->ev_events & EV_CLOSED) {
        if (++nclose == 1)
            res |= EV_CLOSED;
    }

    ...
    // 忽略一些语句

    if (res) {
        void *extra = ((char*)ctx) + sizeof(struct evmap_io);
        /* XXX(niels): we cannot mix edge-triggered and
         * level-triggered, we should probably assert on
         * this. */
        if (evsel->add(base, ev->ev_fd,
            old, (ev->ev_events & EV_ET) | res, extra) == -1)
            return (-1);
        retval = 1;
    }

    ctx->nread = (ev_uint16_t) nread;
    ctx->nwrite = (ev_uint16_t) nwrite;
    ctx->nclose = (ev_uint16_t) nclose;
    LIST_INSERT_HEAD(&ctx->events, ev, ev_io_next);

    return (retval);
}
```

该函数调用 `GET_IO_SLOT_AND_CTOR` 及 `LIST_INSERT_HEAD(&ctx->events, ev, ev_io_next)` 将事件添加到了 event_base 的 `struct event_io_map io` 结构中，并调用 evsel->add 将事件添加到底层的 I/O 多路复用结构，这里还是以 epoll 为例，简单看下 `epoll_nochangelist_add` 接口:

```c
static int
epoll_nochangelist_add(struct event_base *base, evutil_socket_t fd,
    short old, short events, void *p)
{
    struct event_change ch;
    ch.fd = fd;
    ch.old_events = old;
    ch.read_change = ch.write_change = ch.close_change = 0;
    if (events & EV_WRITE)
        ch.write_change = EV_CHANGE_ADD |
            (events & EV_ET);
    if (events & EV_READ)
        ch.read_change = EV_CHANGE_ADD |
            (events & EV_ET);
    if (events & EV_CLOSED)
        ch.close_change = EV_CHANGE_ADD |
            (events & EV_ET);

    return epoll_apply_one_change(base, base->evbase, &ch);
}
```

`epoll_apply_one_change` 将事件添加到 epollop 中的 epfd，注意这里将 event_base 的 evbase 作为 `struct epollop *epollop` 传递给 epoll_apply_one_change，epoll_apply_one_change 则调用 epoll_ctl 来完成事件的添加、修改及删除。

```c
static int
epoll_apply_one_change(struct event_base *base,
    struct epollop *epollop,
    const struct event_change *ch)
{
    struct epoll_event epev;
    int op, events = 0;
    int idx;

    idx = EPOLL_OP_TABLE_INDEX(ch);
    op = epoll_op_table[idx].op;
    events = epoll_op_table[idx].events;

    if (!events) {
        EVUTIL_ASSERT(op == 0);
        return 0;
    }

    if ((ch->read_change|ch->write_change|ch->close_change) & EV_CHANGE_ET)
        events |= EPOLLET;

    memset(&epev, 0, sizeof(epev));
    epev.data.fd = ch->fd;
    epev.events = events;
    if (epoll_ctl(epollop->epfd, op, ch->fd, &epev) == 0) {
        event_debug((PRINT_CHANGES(op, epev.events, ch, "okay")));
        return 0;
    }

    switch (op) {
    case EPOLL_CTL_MOD:
        if (errno == ENOENT) {
            /* If a MOD operation fails with ENOENT, the
             * fd was probably closed and re-opened.  We
             * should retry the operation as an ADD.
             */
            if (epoll_ctl(epollop->epfd, EPOLL_CTL_ADD, ch->fd, &epev) == -1) {
                event_warn("Epoll MOD(%d) on %d retried as ADD; that failed too",
                    (int)epev.events, ch->fd);
                return -1;
            } else {
                event_debug(("Epoll MOD(%d) on %d retried as ADD; succeeded.",
                    (int)epev.events,
                    ch->fd));
                return 0;
            }
        }
        break;
    case EPOLL_CTL_ADD:
        if (errno == EEXIST) {
            /* If an ADD operation fails with EEXIST,
             * either the operation was redundant (as with a
             * precautionary add), or we ran into a fun
             * kernel bug where using dup*() to duplicate the
             * same file into the same fd gives you the same epitem
             * rather than a fresh one.  For the second case,
             * we must retry with MOD. */
            if (epoll_ctl(epollop->epfd, EPOLL_CTL_MOD, ch->fd, &epev) == -1) {
                event_warn("Epoll ADD(%d) on %d retried as MOD; that failed too",
                    (int)epev.events, ch->fd);
                return -1;
            } else {
                event_debug(("Epoll ADD(%d) on %d retried as MOD; succeeded.",
                    (int)epev.events,
                    ch->fd));
                return 0;
            }
        }
        break;
    case EPOLL_CTL_DEL:
        if (errno == ENOENT || errno == EBADF || errno == EPERM) {
            /* If a delete fails with one of these errors,
             * that's fine too: we closed the fd before we
             * got around to calling epoll_dispatch. */
            event_debug(("Epoll DEL(%d) on fd %d gave %s: DEL was unnecessary.",
                (int)epev.events,
                ch->fd,
                strerror(errno)));
            return 0;
        }
        break;
    default:
        break;
    }

    event_warn(PRINT_CHANGES(op, epev.events, ch, "failed"));
    return -1;
}
```

至此，我们完成了事件添加流程的分析。下面来看最后一个 `event_base_dispatch`。

**[event_base_dispatch][event_base_dispatch]**

event_base_dispatch 调用了 event_base_loop，该函数循环等待事件的发生，直到 event_base 结构中没有 pending 状态的事件即退出:

```c
int
event_base_loop(struct event_base *base, int flags)
{
    const struct eventop *evsel = base->evsel;
    struct timeval tv;
    struct timeval *tv_p;
    int res, done, retval = 0;
    struct evwatch_prepare_cb_info prepare_info;
    struct evwatch_check_cb_info check_info;
    struct evwatch *watcher;

    ...
    // 删除了一些语句

    base->running_loop = 1;

    done = 0;

    base->event_gotterm = base->event_break = 0;

    while (!done) {
        base->event_continue = 0;
        base->n_deferreds_queued = 0;

        /* Terminate the loop if we have been asked to */
        if (base->event_gotterm) {
            break;
        }

        if (base->event_break) {
            break;
        }

        res = evsel->dispatch(base, tv_p);

        if (res == -1) {
            event_debug(("%s: dispatch returned unsuccessfully.",
                __func__));
            retval = -1;
            goto done;
        }

        ...
        // 删除了一些语句

        if (N_ACTIVE_CALLBACKS(base)) {
            int n = event_process_active(base);
            if ((flags & EVLOOP_ONCE)
                && N_ACTIVE_CALLBACKS(base) == 0
                && n != 0)
                done = 1;
        } else if (flags & EVLOOP_NONBLOCK)
            done = 1;
    }
    event_debug(("%s: asked to terminate loop.", __func__));

done:
    clear_time_cache(base);
    base->running_loop = 0;

    return (retval);
}
```

event_base_loop 调用 evsel->dispatch 将就绪（active）的事件添加到 base->activequeues 对应优先级的队列中，然后调用 event_process_active 处理这些事件（调用 event_new 指定的回调等），这里我们只关注 dispatch，还是以 epoll 对应的接口为例:

```c
static int
epoll_dispatch(struct event_base *base, struct timeval *tv)
{
    struct epollop *epollop = base->evbase;
    struct epoll_event *events = epollop->events;
    int i, res;
    long timeout = -1;

    ...
    // 删除了一些语句，其中包括超时的处理

    res = epoll_wait(epollop->epfd, events, epollop->nevents, timeout);

    if (res == -1) {
        if (errno != EINTR) {
            event_warn("epoll_wait");
            return (-1);
        }

        return (0);
    }

    event_debug(("%s: epoll_wait reports %d", __func__, res));
    EVUTIL_ASSERT(res <= epollop->nevents);

    for (i = 0; i < res; i++) {
        int what = events[i].events;
        short ev = 0;
#ifdef USING_TIMERFD
        if (events[i].data.fd == epollop->timerfd)
            continue;
#endif

        if (what & EPOLLERR) {
            ev = EV_READ | EV_WRITE;
        } else if ((what & EPOLLHUP) && !(what & EPOLLRDHUP)) {
            ev = EV_READ | EV_WRITE;
        } else {
            if (what & EPOLLIN)
                ev |= EV_READ;
            if (what & EPOLLOUT)
                ev |= EV_WRITE;
            if (what & EPOLLRDHUP)
                ev |= EV_CLOSED;
        }

        if (!ev)
            continue;

        evmap_io_active_(base, events[i].data.fd, ev | EV_ET);
    }

    if (res == epollop->nevents && epollop->nevents < MAX_NEVENT) {
        /* We used all of the event space this time.  We should
           be ready for more events next time. */
        int new_nevents = epollop->nevents * 2;
        struct epoll_event *new_events;

        new_events = mm_realloc(epollop->events,
            new_nevents * sizeof(struct epoll_event));
        if (new_events) {
            epollop->events = new_events;
            epollop->nevents = new_nevents;
        }
    }

    return (0);
}
```

epoll_dispatch 调用 epoll_wait 等待事件的到来，然后循环处理相应的事件并调用 evmap_io_active_ 将对应的事件添加到 base->activequeues 对应优先级的队列中。值得注意的是，这里还会按需扩展 epollop 中事件列表的大小。

至此，libevent 最基本的四个接口分析完毕，下面给出两个用 libevent 实现的 echo server。

*注: 本文只把大体的框架进行了分析，忽略了事件、超时、epoll changelists 等特性，读者可按需阅读相应的源码。*

<h4>案例</h4>

**不使用 bufferevent 的 echo server**

```c
/*
 * libevent echo server example.
 */

#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>

/* For inet_ntoa. */
#include <arpa/inet.h>

/* Required by event.h. */
#include <sys/time.h>

#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <fcntl.h>
#include <errno.h>
#include <err.h>

/* Libevent. */
#include <event2/event.h>
#include <event2/util.h>

/* Port to listen on. */
#define SERVER_PORT 5555

/**
 * This function will be called by libevent when the client socket is
 * ready for reading.
 */
void on_read(int fd, short ev, void *arg)
{
    struct event *client = (struct event *)arg;
    u_char buf[8196];
    int len, wlen;

    len = read(fd, buf, sizeof(buf));
    if (len == 0) {
        /* Client disconnected, remove the read event and the
         * free the client structure. */
        printf("Client disconnected.\n");
                close(fd);
        event_del(client);
        event_free(client);
        return;
    } else if (len < 0) {
        /* Some other error occurred, close the socket, remove
         * the event and free the client structure. */
        printf("Socket failure, disconnecting client: %s",
            strerror(errno));
        close(fd);
        event_del(client);
        event_free(client);
        return;
    }

    /* XXX For the sake of simplicity we'll echo the data write
        * back to the client.  Normally we shouldn't do this in a
        * non-blocking app, we should queue the data and wait to be
        * told that we can write.
        */
    wlen = write(fd, buf, len);
    if (wlen < len) {
        /* We didn't write all our data.  If we had proper
        * queueing/buffering setup, we'd finish off the write
        * when told we can write again.  For this simple case
        * we'll just lose the data that didn't make it in the
        * write.
        */
        printf("Short write, not all data echoed back to client.\n");
    }
}

/**
 * This function will be called by libevent when there is a connection
 * ready to be accepted.
 */
void on_accept(evutil_socket_t fd, short ev, void *arg)
{
    struct event_base *base = (struct event_base *)arg;
    int client_fd;
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);

    /* Accept the new connection. */
    client_fd = accept(fd, (struct sockaddr *)&client_addr, &client_len);
    if (client_fd == -1) {
        warn("accept failed");
        return;
    }

    /* Set the client socket to non-blocking mode. */
    if (evutil_make_socket_nonblocking(client_fd) < 0)
        warn("failed to set client socket non-blocking");

    /* We've accepted a new client, event_new to
     * maintain the state of this client. */
    struct event *new_ev = event_new(base, client_fd, EV_READ|EV_PERSIST, on_read, event_self_cbarg());

    /* Setting up the event does not activate, add the event so it
     * becomes active. */
    event_add(new_ev, NULL);

    printf("Accepted connection from %s\n",
            inet_ntoa(client_addr.sin_addr));
}

int main()
{
    evutil_socket_t listen_fd;
    struct event_base *base;
    struct event *listen_event;        // listener
    struct sockaddr_in listen_addr;

    base = event_base_new();
    if (!base)
        err(1, "new base failed");

    listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0)
        err(1, "listen failed");
    /* Set the socket to non-blocking, this is essential in event
     * based programming with libevent. */
    evutil_make_socket_nonblocking(listen_fd);
    evutil_make_listen_socket_reuseable(listen_fd);

    memset(&listen_addr, 0, sizeof(listen_addr));
    listen_addr.sin_family = AF_INET;
    listen_addr.sin_addr.s_addr = INADDR_ANY;
    listen_addr.sin_port = htons(SERVER_PORT);
    if (bind(listen_fd, (struct sockaddr *)&listen_addr, sizeof(listen_addr)) < 0)
        err(1, "bind failed");
    if (listen(listen_fd, 5) < 0)
        err(1, "listen failed");

    /* We now have a listening socket, we create a read event to
     * be notified when a client connects. */
    listen_event = event_new(base, listen_fd, EV_READ|EV_PERSIST, on_accept, (void *)base);
    event_add(listen_event, NULL);

    /* Start the libevent event loop. */
    event_base_dispatch(base);

    return 0;
}
```

**使用 bufferevent 的 echo server**

```c
/*
 * libevent echo server example using buffered events.
 */

#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <arpa/inet.h>

/* Required by event.h. */
#include <sys/time.h>

#include <stdlib.h>
#include <stdio.h>
#include <string.h>
#include <fcntl.h>
#include <unistd.h>
#include <errno.h>
#include <err.h>

/* Libevent. */
#include <event2/event.h>
#include <event2/util.h>
#include <event2/buffer.h>
#include <event2/bufferevent.h>
#include <event2/bufferevent_struct.h>

/* Port to listen on. */
#define SERVER_PORT 5555

/**
 * Called by libevent when there is data to read.
 */
void buffered_on_read(struct bufferevent *bev, void *arg)
{
    /* Write back the read buffer. It is important to note that
     * bufferevent_write_buffer will drain the incoming data so it
     * is effectively gone after we call it. */
    bufferevent_write_buffer(bev, bev->input);
}

/**
 * Called by libevent when there is an error on the underlying socket
 * descriptor.
 */
void buffered_on_error(struct bufferevent *bev, short what, void *arg)
{
    if (what & BEV_EVENT_EOF) {
        /* Client disconnected, remove the read event and the
         * free the client structure. */
        printf("Client disconnected.\n");
    } else {
        warn("Client socket error, disconnecting.\n");
    }
    bufferevent_free(bev);
    close(bev->ev_read.ev_fd);
}

/**
 * This function will be called by libevent when there is a connection
 * ready to be accepted.
 */
void on_accept(int fd, short ev, void *arg)
{
    struct event_base *base = (struct event_base *)arg;
    int client_fd;
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);
    struct client *client;

    client_fd = accept(fd, (struct sockaddr *)&client_addr, &client_len);
    if (client_fd < 0) {
        warn("accept failed");
        return;
    }

    /* Set the client socket to non-blocking mode. */
    if (evutil_make_socket_nonblocking(client_fd) < 0)
        warn("failed to set client socket non-blocking");

    /* Create the buffered event.
     *
     * The first argument is the file descriptor that will trigger
     * the events, in this case the clients socket.
     *
     * The second argument is the callback that will be called
     * when data has been read from the socket and is available to
     * the application.
     *
     * The third argument is a callback to a function that will be
     * called when the write buffer has reached a low watermark.
     * That usually means that when the write buffer is 0 length,
     * this callback will be called.  It must be defined, but you
     * don't actually have to do anything in this callback.
     *
     * The fourth argument is a callback that will be called when
     * there is a socket error.  This is where you will detect
     * that the client disconnected or other socket errors.
     *
     * The fifth and final argument is to store an argument in
     * that will be passed to the callbacks.  We store the client
     * object here.
     */
    struct bufferevent *bev = bufferevent_socket_new(base, client_fd, BEV_OPT_CLOSE_ON_FREE);
    bufferevent_setcb(bev, buffered_on_read, NULL, buffered_on_error, NULL);
    /* We have to enable it before our callbacks will be
     * called. */
    bufferevent_enable(bev, EV_READ);

    printf("Accepted connection from %s\n",
        inet_ntoa(client_addr.sin_addr));
}

int main()
{
    evutil_socket_t listen_fd;
    struct event_base *base;
    struct event *listen_event;        // listener
    struct sockaddr_in listen_addr;

    base = event_base_new();
    if (!base)
        err(1, "new base failed");

    /* Create our listening socket. */
    listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0)
        err(1, "listen failed");
    memset(&listen_addr, 0, sizeof(listen_addr));
    listen_addr.sin_family = AF_INET;
    listen_addr.sin_addr.s_addr = INADDR_ANY;
    listen_addr.sin_port = htons(SERVER_PORT);
    if (bind(listen_fd, (struct sockaddr *)&listen_addr,
        sizeof(listen_addr)) < 0)
        err(1, "bind failed");
    if (listen(listen_fd, 5) < 0)
        err(1, "listen failed");

    evutil_make_listen_socket_reuseable(listen_fd);

    /* Set the socket to non-blocking, this is essential in event
     * based programming with libevent. */
    if (evutil_make_socket_nonblocking(listen_fd) < 0)
        err(1, "failed to set server socket to non-blocking");

    /* We now have a listening socket, we create a read event to
     * be notified when a client connects. */
    listen_event = event_new(base, listen_fd, EV_READ|EV_PERSIST, on_accept, (void *)base);
    event_add(listen_event, NULL);

    /* Start the event loop. */
    event_base_dispatch(base);

    return 0;
}
```

编译运行详见 [libevent-examples][libevent-examples]。

<br>
<span class="post-meta">
Reference:
</span>
<br>
<span class="post-meta">
1 [Fast portable non-blocking network programming with Libevent][libevent_book].<br>
</span>

[libevent]: https://libevent.org/
[libevent_book]: http://www.wangafu.net/~nickm/libevent-book/
[unp]: http://unpbook.com/
[tcpservpoll01]: https://github.com/zhjwpku/awesome-courses-labs/blob/master/unpv13e/tcpcliserv/tcpservpoll01.c
[tcpservepoll01]: https://github.com/zhjwpku/awesome-courses-labs/pull/3/files
[event_base]: https://github.com/libevent/libevent/blob/release-2.1.12-stable/event-internal.h#L208
[event_base_new]: https://github.com/libevent/libevent/blob/release-2.1.12-stable/event.c#L520
[event_config_new]: https://github.com/libevent/libevent/blob/release-2.1.12-stable/event.c#L520
[eventops]: https://github.com/libevent/libevent/blob/release-2.1.12-stable/event.c#L103
[epoll_init]: https://github.com/libevent/libevent/blob/release-2.1.12-stable/epoll.c#L117
[event_new]: https://github.com/libevent/libevent/blob/release-2.1.12-stable/event.c#L2208
[event_add]: https://github.com/libevent/libevent/blob/release-2.1.12-stable/event.c#L2484
[event_base_dispatch]: https://github.com/libevent/libevent/blob/release-2.1.12-stable/event.c#L1815
[libevent-examples]: https://github.com/zhjwpku/libevent-examples

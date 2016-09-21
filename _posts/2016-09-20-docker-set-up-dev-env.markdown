---
layout: post
title: 搭建Docker开发环境
date: 2016-09-20 18:00:00 +0800
categories: docker
tags:
- docker
- dev
---

本文参照Docker官网的[Work with a development container][set-up-dev-env]搭建了Docker的开发环境，据说Docker Engine的核心开发团队就是这么个开发流程:)

*大家也可以使用[DockerHub][dockerhub]上的官方docker镜像来搭建。*

`docker`源码仓库中有一个`Dockerfile`文件，定义了Docker的开发环境，包括系统软件库、Go环境以及Go的依赖等。

<h4>Task 1. 删除镜像和容器</h4>
Docker开发者一般会运行最新的稳定版Docker release，删除停止的容器以及没用的images虽然不是必要的任务，但这通常是一个好习惯。

{% highlight shell %}
$ docker rm $(docker ps -a -q)
$ docker rmi -f $(docker images -q -a dangling=true)
{% endhighlight %}

<h4>Task 2. 运行一个开发容器</h4>
{% highlight shell %}
# 进入docker仓库
$ cd ~/go/src/github.com/docker/docker
$ git branch develop
$ git checkout develop
# 使用make来构建docker的开发环境，这个过程比较漫长
$ make shell
...output snipped...
Successfully built 848f3059bdc3
docker run --rm -i --privileged -e BUILDFLAGS -e KEEPBUNDLE -e DOCKER_BUILD_GOGC -e DOCKER_BUILD_PKGS -e DOCKER_CLIENTONLY -e DOCKER_DEBUG -e DOCKER_EXPERIMENTAL -e DOCKER_GITCOMMIT -e DOCKER_GRAPHDRIVER=devicemapper -e DOCKER_INCREMENTAL_BINARY -e DOCKER_REMAP_ROOT -e DOCKER_STORAGE_OPTS -e DOCKER_USERLANDPROXY -e TESTDIRS -e TESTFLAGS -e TIMEOUT -v "home/zhjwpku/go/src/github.com/docker/docker/bundles:/go/src/github.com/docker/docker/bundles" -t "docker-dev:develop" bash
root@9f457fc6b4a9:/go/src/github.com/docker/docker# exit
$ 
{% endhighlight %}

这样我们就得到了一个可用的docker开发环境镜像，可以使用`docker images`查看，并使用如下命令重新启动docker开发容器：
{% highlight shell %}
# 在docker仓库目录下运行如下命令
$ docker run -v `pwd`:/go/src/github.com/docker/docker --privileged -i -t docker-dev:develop bash
root@948bd6956b4c:/go/src/github.com/docker/docker#
{% endhighlight %}

这样就把docker源码仓库挂载到了开发环境的容器中，对docker源码仓库中的任何改动都会映射到开发环境容器中。

<h4>Build Docker源码</h4>
{% highlight shell %}
root@948bd6956b4c:/go/src/github.com/docker/docker# hack/make.sh binary
...output snipped...
root@948bd6956b4c:/go/src/github.com/docker/docker# cp bundles/1.13.0-dev/binary-client/docker* /usr/bin
root@948bd6956b4c:/go/src/github.com/docker/docker# cp bundles/1.13.0-dev/binary-daemon/docker* /usr/bin
# 运行Docker Engine Daemon, -D打开Debug模式
root@948bd6956b4c:/go/src/github.com/docker/docker# docker daemon -D&
...output snipped...
# 查看Docker版本
root@948bd6956b4c:/go/src/github.com/docker/docker# docker --version
Docker version 1.13.0-dev, build 45a8f68
{% endhighlight %}

至此，我们可以像Docker Engine核心团队一样来开发Docker了:)

[set-up-dev-env]: https://docs.docker.com/opensource/project/set-up-dev-env/
[dockerhub]: https://hub.docker.com/_/docker/

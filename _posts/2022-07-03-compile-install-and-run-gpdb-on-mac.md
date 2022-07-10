---
layout: post
title: 在 Mac 上编译安装并运行 Greenplum
date: 2022-07-03 00:00:00 +0800
tags:
- postgres
- gpdb
---

记录在 MacOS 上编译、安装、运行 Greenplum 的步骤。

**下载代码**

```shell
git clone https://github.com/greenplum-db/gpdb.git
cd gpdb
```

**安装依赖**

```shell
./README.macOS.bash
source ~/.bashrc

## 如果遇到 Error: unable to import module: No module named 'pgdb' 报错安装 PyGreSQL，而非 pgdb
pip3 install PyGreSQL
pip3 install psutil
```

**配置 ssh**

```shell
mkdir -p "$HOME/.ssh"

cat >> ~/.bash_profile << EOF

# Allow ssh to use the version of python in path, not the system python
# BEGIN SSH agent
# from http://stackoverflow.com/questions/18880024/start-ssh-agent-on-login/18915067#18915067
SSH_ENV="\$HOME/.ssh/environment"
# Refresh the PATH per new session
sed -i .bak '/^PATH/d' \${SSH_ENV}
echo "PATH=\$PATH" >> \${SSH_ENV}

function start_agent {
    echo "Initialising new SSH agent..."
    /usr/bin/ssh-agent | sed 's/^echo/#echo/' > "\${SSH_ENV}"
    echo succeeded
    chmod 600 "\${SSH_ENV}"
    source "\${SSH_ENV}" > /dev/null
    /usr/bin/ssh-add;
}

# Source SSH settings, if applicable
if [ -f "\${SSH_ENV}" ]; then
    . "\${SSH_ENV}" > /dev/null
    ps -ef | grep \${SSH_AGENT_PID} 2>/dev/null | grep ssh-agent$ > /dev/null || {
        start_agent;
    }
else
    start_agent;
fi

[ -f ~/.bashrc ] && source ~/.bashrc
# END SSH agent
EOF

source ~/.bash_profile
sudo tee -a /etc/ssh/sshd_config << EOF
# Allow ssh to use the version of python in path, not the system python
PermitUserEnvironment yes
EOF
```

确保能够从本机免密登录:

```shell
ssh `hostname`
```

**编译安装**

```shell
./configure --with-perl --with-python --with-libxml --with-gssapi --enable-debug --disable-gpfdist
make -j8
sudo make install

# Bring in greenplum environment into your running shell
source /usr/local/gpdb/greenplum_path.sh
```

**启动 demo 集群**

```shell
# 查看端口号是否被占用
sudo lsof -PiTCP -sTCP:LISTEN

# 选择一个端口号启动集群
cd gpAux/gpdemo
WITH_MIRRORS=false WITH_STANDBY=false PORT_BASE=5555 make cluster

# 设置环境变量
source gpdemo-env.sh
```

**删除 demo 集群**

```shell
make clean
```

**启停集群**

在使用集群的过程中，并不需要频繁地创建或是删除集群，应该使用 gpdb 提供的工具来启停集群:

```shell
# 停止集群
gpstop -a

# 启动集群
gpstart -a

# 查看集群状态
gpstate -s
```

**Troubleshooting**

`.bashrc` 中使用 `ulimit -n 65536 65536` 设置了文件描述符的个数和文件的大小，但在集群启动的时候遇到了 *Child process was terminated by signal 25, File size limit exceeded* 的错误，改为 `ulimit -n 65536 unlimited` 后解决问题。另外在解决问题的过程中还进行了如下设置:

```shell
launchctl limit maxfiles
sudo launchctl limit maxfiles 65536 1048576
```

但不确定是否跟问题的解决相关。

<br>
<span class="post-meta">
References:
</span>
<br>
<span class="post-meta">
1 [gpdb/README.macOS.md](https://github.com/greenplum-db/gpdb/blob/master/README.macOS.md)<br>
</span>


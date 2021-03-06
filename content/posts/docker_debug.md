---
title: Docker 调试技巧
date: 2016-12-17T22:30:27+08:00
categories:
- docker
tags:
- docker
---
### 『重用』容器名
但我们在编写/调试Dockerfile的时候我们经常会重复之前的command，比如这种`docker run --name jstorm-zookeeper zookeeper:3.4`，然后就容器名就冲突了。
```shell
$ docker run --name jstorm-zookeeper zookeeper:3.4
...
$ docker run --name jstorm-zookeeper zookeeper:3.4
docker: Error response from daemon: Conflict. The name "/jstorm-zookeeper" is already in use by container xxxxxxxxx
```

可以在运行 `docker run` 时候加上`--rm` flag, 容器将在退出之后销毁。无需手动`docker rm CONTAINER`

```shell
$ docker run --name jstorm-zookeeper zookeeper:3.4 --rm

# reuse 
$ docker create --name jstorm-zookeeper zookeeper:3.4
$ docker start jstorm-zookeeper

# no error
```

<!-- more -->

### debug Dockerfile

在写 `Dockerfile` 的时候，通常并不会一气呵成。有的时候容器启动就crash 直接退出，有的时候build image 就会失败，或者想验证Dockerfile是否符合预期，我们经常要debug Dockerfile。

如果build 失败可以直接 查看stdout的错误信息，拆分指令，重新build。

### `logs` 查看 stdout
所有容器内写到stdout的内容都会被捕获到host中的一个history文件中, 可以通过 `docker logs CONTAINER` 查看。
```
$ docker run -d --name jstorm-zookeeper zookeeper:3.4

$ docker logs jstorm-zookeeper
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
2016-12-18 05:55:27,717 [myid:] - INFO  [main:QuorumPeerConfig@124] - Reading configuration from: /conf/zoo.cfg
2016-12-18 05:55:27,725 [myid:] - INFO  [main:DatadirCleanupManager@78] - autopurge.snapRetainCount set to 3
2016-12-18 05:55:27,725 [myid:] - INFO  [main:DatadirCleanupManager@79] - autopurge.purgeInterval set to 0
2016-12-18 05:55:27,726 [myid:] - INFO  [main:DatadirCleanupManager@101] - Purge task is not scheduled.
2016-12-18 05:55:27,728 [myid:] - WARN  [main:QuorumPeerMain@113] - Either no config or no quorum defined in config, running  in standalone mode
2016-12-18 05:55:27,746 [myid:] - INFO  [main:QuorumPeerConfig@124] - Reading configuration from: /conf/zoo.cfg
2016-12-18 05:55:27,747 [myid:] - INFO  [main:ZooKeeperServerMain@96] - Starting server
2016-12-18 05:55:27,766 [myid:] - INFO  [main:Environment@100] - Server environment:zookeeper.version=3.4.9-1757313, built on 08/23/2016 06:50 GMT
2016-12-18 05:55:27,766 [myid:] - INFO  [main:Environment@100] - Server environment:host.name=dbc742dd5688
2016-12-18 05:55:27,767 [myid:] - INFO  [main:Environment@100] - Server environment:java.version=1.8.0_111-internal
```
即使是容器已经退出的也可以看到，所以可以通过这种方式来分析非预期的退出。这些文件一直保存着，直到通过`docker rm`把容器删除。文件的具体路径可以通过`docker inspect CONTAINER` 获得。
（然后osx上你并找不到这些文件，因为其实osx的docker实际是运行在"VM"中，具体就不展开了，但是可以通过 `screen ~/Library/Containers/com.docker.docker/Data/com.docker.driver.amd64-linux/tty` touch上"VM"的tty）

在使用`docker logs` 的时候加一些参数来过滤log，默认输出所有log。
```
Options:
      --details        Show extra details provided to logs
  -f, --follow         Follow log output
      --help           Print usage
      --since string   Show logs since timestamp
      --tail string    Number of lines to show from the end of the logs (default "all")
  -t, --timestamps     Show timestamps
```

### `attach` 实时查看stdout
如果你想实时查看容器的输出你可以用 `docker attach CONTAINER` 命令。

默认会绑定stdin，代理signals， 所以如果你 `ctrl-c` 容器通常会退出。很多时候大家并不想这样，只是想分离开，可以`ctrl-p ctrl-q`。

### 执行任意command
可以通过`docker exec CONTAINER COMMAND`，来在容器内执行任意 command，比如 cat 一些东西来debug。

```shell
$ docker run -d --name jstorm-zookeeper zookeeper:3.4

$ docker exec jstorm-zookeeper java -version
openjdk version "1.8.0_111-internal"
OpenJDK Runtime Environment (build 1.8.0_111-internal-alpine-r0-b14)
OpenJDK 64-Bit Server VM (build 25.111-b14, mixed mode)
```

也可以直接通过 exec 在容器内启动一个 shell 更方便地调试容器，不必一条条执行`docker exec`。

```
$ docker exec -it jstorm-zookeeper /bin/bash
bash-4.3# pwd
/zookeeper-3.4.9
```

`docker exec` 只能在正在运行的容器上使用，如果已经停止了退出了就不行了，就只好用 `docker logs` 了。

### 重写`entrypoint`和`cmd`

每个Docker镜像都有 `entrypoint` 和 `cmd` , 可以定义在 `Dockerfile` 中，也可以在运行时指定。这两个概念很容易混淆，而且它们的试用方式也不同。

`entrypoint` 比 `cmd` 更"高级"，`entrypoint` 作为容器中pid为1的进程运行（docker不是虚拟机，只是隔离的进程。真正的linux中pid为1的是`init`）。
`cmd` 只是 `entrypoint`的参数:

```
<ENTRYPOINT> "<CMD>"
```

当我们没有指定 `entrypoint` 时缺省为 `/bin/sh -c`。所以其实 `entrypoint` 是真正表达这个docker应该干什么的，通常大家有一个shell 脚本来代理。
`entrypoint` 和 `cmd` 都可以在运行的时候更改，通过更改来看这样设置`entrypoint`是否优雅合理。

```shell
$ docker run -it --name jstorm-zookeeper --entrypoint /bin/bash zookeeper:3.4
bash-4.3# top
Mem: 320212K used, 1725368K free, 89112K shrd, 35532K buff, 130532K cached
CPU:   0% usr   0% sys   0% nic 100% idle   0% io   0% irq   0% sirq
Load average: 0.20 0.06 0.02 5/195 7
  PID  PPID USER     STAT   VSZ %VSZ CPU %CPU COMMAND
    1     0 root     S     6220   0%   0   0% /bin/bash
    7     1 root     R     1516   0%   2   0% top
```
任何 `docker run` 命令中在image名后的内容都作为`cmd`的内容传给 `entrypoint`当参数。


### 暂停容器
使用 `docker pause` 可以暂停容器中所有进程。这非常有用。

```shell
$ docker run -d --name jstorm-zookeeper zookeeper:3.4 && sleep 0.1 && docker pause jstorm-zookeeper && docker logs jstorm-zookeeper
a24405a53ddd9b7d94d9e77fe2b5a67639a251d681aa2f34fcb0cc96f347ba48
jstorm-zookeeper
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
2016-12-18 16:17:47,720 [myid:] - INFO  [main:QuorumPeerConfig@124] - Reading configuration from: /conf/zoo.cfg
2016-12-18 16:17:47,730 [myid:] - INFO  [main:DatadirCleanupManager@78] - autopurge.snapRetainCount set to 3
2016-12-18 16:17:47,730 [myid:] - INFO  [main:DatadirCleanupManager@79] - autopurge.purgeInterval set to 0
2016-12-18 16:17:47,730 [myid:] - INFO  [main:DatadirCleanupManager@101] - Purge task is not scheduled.
2016-12-18 16:17:47,731 [myid:] - WARN  [main:QuorumPeerMain@113] - Either no config or no quorum defined in config, running  in standalone mode
2016-12-18 16:17:47,757 [myid:] - INFO  [main:QuorumPeerConfig@124] - Reading configuration from: /conf/zoo.cfg
2016-12-18 16:17:47,757 [myid:] - INFO  [main:ZooKeeperServerMain@96] - Starting server

$ docker unpause jstorm-zookeeper && docker logs jstorm-zookeeper
jstorm-zookeeper
ZooKeeper JMX enabled by default
Using config: /conf/zoo.cfg
2016-12-18 16:17:47,720 [myid:] - INFO  [main:QuorumPeerConfig@124] - Reading configuration from: /conf/zoo.cfg
2016-12-18 16:17:47,730 [myid:] - INFO  [main:DatadirCleanupManager@78] - autopurge.snapRetainCount set to 3
2016-12-18 16:17:47,730 [myid:] - INFO  [main:DatadirCleanupManager@79] - autopurge.purgeInterval set to 0
2016-12-18 16:17:47,730 [myid:] - INFO  [main:DatadirCleanupManager@101] - Purge task is not scheduled.
2016-12-18 16:17:47,731 [myid:] - WARN  [main:QuorumPeerMain@113] - Either no config or no quorum defined in config, running  in standalone mode
2016-12-18 16:17:47,757 [myid:] - INFO  [main:QuorumPeerConfig@124] - Reading configuration from: /conf/zoo.cfg
2016-12-18 16:17:47,757 [myid:] - INFO  [main:ZooKeeperServerMain@96] - Starting server
2016-12-18 16:18:09,039 [myid:] - INFO  [main:Environment@100] - Server environment:zookeeper.version=3.4.9-1757313, built on 08/23/2016 06:50 GMT
2016-12-18 16:18:09,040 [myid:] - INFO  [main:Environment@100] - Server environment:host.name=a24405a53ddd
2016-12-18 16:18:09,040 [myid:] - INFO  [main:Environment@100] - Server environment:java.version=1.8.0_111-internal
2016-12-18 16:18:09,040 [myid:] - INFO  [main:Environment@100] - Server environment:java.vendor=Oracle Corporation
2016-12-18 16:18:09,040 [myid:] - INFO  [main:Environment@100] - Server environment:java.home=/usr/lib/jvm/java-1.8-openjdk/jre
2016-12-18 16:18:09,040 [myid:] - INFO  [main:Environment@100] - Server environment:java.class.path=/zookeeper-3.4.9/bin/../build/classes:/zookeeper-3.4.9/bin/../build/lib/*.jar:/zookeeper-3.4.9/bin/../lib/slf4j-log4j12-1.6.1.jar:/zookeeper-3.4.9/bin/../lib/slf4j-api-1.6.1.jar:/zookeeper-3.4.9/bin/../lib/netty-3.10.5.Final.jar:/zookeeper-3.4.9/bin/../lib/log4j-1.2.16.jar:/zookeeper-3.4.9/bin/../lib/jline-0.9.94.jar:/zookeeper-3.4.9/bin/../zookeeper-3.4.9.jar:/zookeeper-3.4.9/bin/../src/java/lib/*.jar:/conf:
2016-12-18 16:18:09,040 [myid:] - INFO  [main:Environment@100] - Server environment:java.library.path=/usr/lib/jvm/java-1.8-openjdk/jre/lib/amd64/server:/usr/lib/jvm/java-1.8-openjdk/jre/lib/amd64:/usr/lib/jvm/java-1.8-openjdk/jre/../lib/amd64:/usr/java/packages/lib/amd64:/usr/lib64:/lib64:/lib:/usr/lib
2016-12-18 16:18:09,041 [myid:] - INFO  [main:Environment@100] - Server environment:java.io.tmpdir=/tmp
2016-12-18 16:18:09,041 [myid:] - INFO  [main:Environment@100] - Server environment:java.compiler=<NA>
2016-12-18 16:18:09,043 [myid:] - INFO  [main:Environment@100] - Server environment:os.name=Linux
2016-12-18 16:18:09,043 [myid:] - INFO  [main:Environment@100] - Server environment:os.arch=amd64
2016-12-18 16:18:09,044 [myid:] - INFO  [main:Environment@100] - Server environment:os.version=4.4.27-moby
2016-12-18 16:18:09,044 [myid:] - INFO  [main:Environment@100] - Server environment:user.name=zookeeper
2016-12-18 16:18:09,044 [myid:] - INFO  [main:Environment@100] - Server environment:user.home=/home/zookeeper
2016-12-18 16:18:09,044 [myid:] - INFO  [main:Environment@100] - Server environment:user.dir=/zookeeper-3.4.9
2016-12-18 16:18:09,057 [myid:] - INFO  [main:ZooKeeperServer@815] - tickTime set to 2000
2016-12-18 16:18:09,057 [myid:] - INFO  [main:ZooKeeperServer@824] - minSessionTimeout set to -1
2016-12-18 16:18:09,058 [myid:] - INFO  [main:ZooKeeperServer@833] - maxSessionTimeout set to -1
2016-12-18 16:18:09,076 [myid:] - INFO  [main:NIOServerCnxnFactory@89] - binding to port 0.0.0.0/0.0.0.0:2181
```

### `top` 和 `stats` 获得容器中进程的状态
`docker top CONTAINER` 和在容器里执行 `top` 的效果类似。

```shell
$ docker top jstorm-zookeeper
PID    USER      TIME  COMMAND
24593  dockrema  0:01  /usr/lib/jvm/java-1.8-openjdk/jre/bin/java -Dzookeeper.log.dir=. .....

$ docker stats jstorm-zookeeper
CONTAINER           CPU %     MEM USAGE / LIMIT       MEM %   NET I/O        BLOCK I/O   PIDS
jstorm-zookeeper    0.00%     24.86 MiB / 1.951 GiB   1.24%   648 B / 648 B  0 B / 0 B   20
```

### 通过 `inspect` 查看容器的详细信息
`docker inspect CONTAINER` 饭后镜像和容器的详细信息。比如：

- State —— 容器的当先状态
- LogPath —— history(stdout) file 的路径
- Config.Env —— 环境变量
- NetworkSettings.Ports —— 端口的映射关系

环境变量非常有用很多问题都是环境变量引起的。

### `history` 查看 image layers
可以看到各层创建的指令，大小和哈希。可以用来检查这个image是否符合你的预期。













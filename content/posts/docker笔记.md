---
title:  docker 笔记
description: notes about docker
date: 2019-12-14
categories: [
    "docker",
]
tags: ["docker", "network"]
---

brief
<!--more-->

### 进入 docker 的 network namespace 
docker 的网络隔离用了 linux 的 namespace，这里就不多做阐述了。现在正常的生产环境中的镜像为了尽量的轻，可能不会去安装额外的无用的程序，比如 netstat, curl, wget 等工具，但是有时候需要去排查网络问题进去容器的时候发现没有这些工具又很麻烦，下面这个方式就是在主机上直接进入容器的 network namespace 去进行排查。

假设你有一个 webserver 服务跑在容器上，你现在容器所在的主机上：

- 找到容器的 container id.
```
docker ps | grep webserver
```
显示如下
![内容](/images/docker_note/docker_id.png)

- 从上图中可以知道容器的 id, 然后通过这个 id 拿到 pid.
```
docker inspect ${container_id} --format='{{.State.Pid}}'
```
显示如下
![内容](/images/docker_note/pid.png)

- 找到 pid 后就可以直接进入 network namespace 了
```
nsenter -t ${pid} -n
```
这时候就进入了容器的 network namesapce 了，使用 `ifconfig` 可以看到当前的网卡信息
![图片](/images/docker_note/ifconfig.png)

title: 搭建本地私有docker仓库
date: 2016-06-16 16:37:12
tags:
- docker
- registry
categories: 运维
---

在[上一篇](http://blog.kazaff.me/2016/06/16/%E5%B0%9D%E8%AF%95%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90--%E7%AC%AC%E4%B8%80%E7%89%88/)中，我们已经把gitlabCI生成的镜像推送到[灵雀云](https://hub.alauda.cn/repos/kazaff/hello-kazaff)上面。不过这还不够，毕竟真实项目的镜像还是需要放在内网环境下才能避免不必要的麻烦和确保良好的下载速度（说白了就是没钱买私有镜像仓库~~）。

首先，我们先把之前的那个demo项目的`.gitlab-ci.yml`修改一下：
```
image: docker:latest

services:
- docker:dind

build:image:
  stage: build
  script:
    - docker build -t 172.17.0.1:5000/kazaff/hello-kazaff .
    - docker push 172.17.0.1:5000/kazaff/hello-kazaff
  tags:
    - docker

```
注意，这次我们push到了自己搭建的Registry中。本以为只要使用官方提供的[registry](https://hub.docker.com/r/library/registry)直接run一个docker容器就搞定了，要真那么顺利也就写这篇日志的必要了！

按照官方文档，有两种方法来搭建私有仓库，当然，第二种依赖亚马逊S3的路子我们pass了，憋说话，我是穷逼。

而直接执行第一种方法给的命令，等着你的就是悲剧。首先，从官方下方的留言区可以看出，貌似`latest`并不是指向最新的registry版本，而且我这边pull镜像的时候也总登录不上dockerhub，还是老办法，使用[灵雀云](https://hub.alauda.cn/repos/library/registry)，我并非在给它打广告，确实能找到最开放的国内镜像库了：

```
docker pull index.alauda.cn/library/registry:2.4.1
```
**注意，这次一定要带上tag号啊~**

然后我们直接执行：
```
docker run -d -p 5000:5000 -v /srv/docker/registry:/var/lib/registry --restart=always --name registry
```
这次，就应该搭建好了，如果你想让物理机环境下也能访问，记得依然要做虚拟机的端口映射哦~

然后我们只需要向我们自己的gitlab提交最新版本的项目文件，即可触发build了~

你可能会碰见比我复杂的多的问题，这里有一篇[文章](http://tonybai.com/2016/02/26/deploy-a-private-docker-registry/)总结的很全面，推荐阅读。

这样，我们就基本上搭建了一个自娱自乐的CI/CD环境，不过，想直接投入团队使用还差太多，后面我会继续改善和丰满该工作流。不过今天就先到这里~

title: 搭建本地私有docker仓库
date: 2016-06-16 16:37:12
tags:
- docker
- registry
categories: 运维
---

在[上一篇](http://blog.kazaff.me/2016/06/16/%E5%B0%9D%E8%AF%95%E6%8C%81%E7%BB%AD%E9%9B%86%E6%88%90--%E7%AC%AC%E4%B8%80%E7%89%88/)中，我们已经把gitlabCI生成的镜像推送到[灵雀云](https://hub.alauda.cn/repos/kazaff/hello-kazaff)上面。不过这还不够，毕竟真实项目的镜像还是需要放在内网环境下才能避免不必要的麻烦和确保良好的下载速度（说白了就是没钱买私有镜像仓库~~）。
<!--more-->
首先，我们先把之前的那个demo项目的`.gitlab-ci.yml`修改一下：
```
build:image:
  stage: build
  script:
    - docker build -t 172.17.0.1:5000/kazaff/hello-kazaff .
    - docker push 172.17.0.1:5000/kazaff/hello-kazaff
  tags:
    - shell

```
注意，这次我们push到了自己搭建的Registry中。本以为只要使用官方提供的[registry](https://hub.docker.com/r/library/registry)直接run一个docker容器就搞定了，要真那么顺利也就写这篇日志的必要了！
还有和上一篇文章中使用的Runner方式不同，眼尖的童鞋应该注意到上面新的`.gitlab-ci.yml`中我们把`tags`由原先的`docker`改成了`shell`，后面会说原因，至于如何创建一个executor是shell类型的Runner，上一篇我也介绍过了。

按照官方文档，有两种方法来搭建私有仓库，当然，第二种依赖亚马逊S3的路子我们pass了，憋说话，我是穷逼。

而直接执行第一种方法给的命令，等着你的就是悲剧。首先，从官方下方的留言区可以看出，貌似`latest`并不是指向最新的registry版本，而且我这边pull镜像的时候也总登录不上dockerhub，还是老办法，使用[灵雀云](https://hub.alauda.cn/repos/library/registry)，我并非在给它打广告，确实能找到最开放的国内镜像库了：

```
docker pull index.alauda.cn/library/registry:2.4.1
```
**注意，这次一定要带上tag号啊~**

然后我们直接执行：
```
docker run -d -p 5000:5000 -v /srv/docker/registry:/var/lib/registry --restart=always --name registry index.alauda.cn/library/registry:2.4.1
```
这次，就应该搭建好了，如果你想让物理机环境下也能访问，记得依然要做虚拟机的端口映射哦~

然后我们只需要向我们自己的gitlab提交最新版本的项目文件，即可触发build了~

你可能会碰见比我复杂的多的问题，这里有一篇[文章](http://tonybai.com/2016/02/26/deploy-a-private-docker-registry/)总结的很全面，推荐阅读。

这样，我们就基本上搭建了一个自娱自乐的CI/CD环境，不过，想直接投入团队使用还差太多，后面我会继续改善和丰满该工作流。不过今天就先到这里~

### 补充

之前测试宿主机直接push镜像时，并没有碰到问题，可在Runner中push时肯定不能使用localhost，一旦使用registry所在机器的ip地址时，就会碰到下面的问题：
```
tls: oversized record received with length 20527
```
按照上面的帖子给的方法貌似没有搞定，不过换了[另外一个办法](https://www.linkedin.com/pulse/starting-docker-registry-vinay-thakur)：
```
vi /lib/systemd/system/docker.service
```
该文件中添加`insecure-registry`设置后的样子大概如下：
```
[Unit]
Description=Docker Application Container Engine
Documentation=https://docs.docker.com
After=network.target docker.socket
Requires=docker.socket

[Service]
Type=notify
ExecStart=/usr/bin/docker daemon --insecure-registry=172.17.0.1:5000 -H fd://
MountFlags=slave
LimitNOFILE=1048576
LimitNPROC=1048576
LimitCORE=infinity
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
```
然后我们重启一下docker：
```
systemctl daemon-reload
service docker restart
```
搞定，注意修改成你的registry的ip哦~

此外，我也把项目中所有使用到的镜像都改成了私有镜像仓库地址，速度杠杠的。不过，在下载dockerhub镜像时碰到一个小插曲，总提示我无法登录帐号！查了不少帖子，官方只是说修复了此问题，但我还是无法登录咋整？

后来发现，原来可以试试单独执行：
```
docker login
```
提示你输入username时，不要输入邮箱地址，而是输入你在dockerhub后台设置的username，就能登录了。。。

别高兴太早，我早就暗示过你们，过程是坎坷的！前面我提到过，我们新建了一个Runner，executor类型是shell，为什么呢？之前docker类型的好好的不是么？

前面我们不是提到私有仓库证书的事儿么？虽然我们已经将宿主机的docker环境关闭了TLS，不过别忘了我们的Runner可是docker-in-docker啊~我找了好多地方，都没有提供如何使Runner启动的那个docker也关闭TLS，否则我们依然会碰到上面的那个问题！~恳请知道解决方案的童鞋留言帮助我升级！~

当然，后面我也会为我们的私有镜像仓库搭建TLS环境，到时候自然会完美解决这个问题！暂时让大家失望了，抱歉~小哥我真的是尽力了，不信看图：
![](http://pic.yupoo.com/kazaff/FDdXoiT8/medish.jpg)

依旧别高兴太早，还记得我的docker环境么？虽然我们解决了宿主机下使用私有仓库的TLS问题，但我的物理机是window啊，我使用docker toolbox安装的docker依然无法使用我们的私有仓库，唉，这就是坎坷。不过知道了问题的本质就好办了，只需要看一下[win环境下如何设置insecure-registry](http://stackoverflow.com/questions/30654306/allow-insecure-registry-in-host-provisioned-with-docker-machine)即可：

```
vi ~/.docker/machine/machines/dev/config.json
```
找到文件中下面这段配置按需修改即可：
```
"EngineOptions": {
        "InsecureRegistry": [
            "192.168.1.23:5000"
        ],
    }
```
注意，这里我填写的是私有镜像仓库的宿主机（ubuntu）的端口通过virtualbox网络配置映射到物理机对应端口后，使用的物理机地址，原因你应该已经猜到了，我就不啰嗦了。

总算踉踉跄跄配好了个粗狂版~

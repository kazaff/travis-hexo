title: docker for win
date: 2016-06-13 09:00:00
tags:
- docker
categories: 运维
---

我所在的项目，小伙伴们都在fire in coding，我也不能闲着，我开始为项目的持续交付进行思考和设计。

第一个要解决的问题，就是开发环境，测试环境和线上环境的一致，那不用说，今天最流行的做这个事儿的应该就是docker了吧？！
<!--more-->
我的小mac还在“医院”躺着，而且考虑到同事都是win用户，所以我从docker for win下手，毕竟不希望一下子挑战大家的所有习惯。

作为docker新手的我，花了2天的时间过了一遍手册，然后就开始了实测！下面记录一下我在实际测试的时候碰到的一些小坑。


### 如何使用dockerfile

这在linux环境下可能不是啥问题，直接根据手册提供的命令即可：

```
$ sudo docker build -t myrepo/myapp /tmp/test1/
```
可在win环境下，你首先会碰到无法定位win系统下dockerfile文件的问题，不过docker box已经帮你做了大量的工作（和一些历史教程相比容易好多），它将我们的c:盘直接映射到/c位置，也就是说在`Docker Quickstart Terminal`提供的shell环境下，直接就可以访问c:盘下的文件结构。

接下来我们就可以执行上面的build命令了，不过你会失望的看到下面的这个error:

```
Error: $ docker build -t kz/test . unable to prepare context: unable to evaluate symlinks in Dockerfile path: GetFileAttributesEx C:\Users\Kazaff\Test\testdocker\Dockerfile: The system cannot find the file specified.
```
唉，就知道不会那么顺利，gg了一下，这个问题已经有了不错的[答案](http://stackoverflow.com/questions/35511604/docker-unable-to-prepare-context-unable-to-evaluate-symlinks-in-dockerfile-pat)，我们只需要加一个-f即可：

```
docker build -t kz/test -f ./myDockerFile .
```

**注意，最后还有一个"."，而且我们执行命令时，终端上下文已经是在目标dockerfile所谓的文件夹中了！**

这样你就可以在win下正常build dockerfile了。

还有，记得给你要RUN的命令加上`-y`参数，避免运行命令时的人机交互动作。


### docker compose

虽然很多教程上提到win下的docker需要手动安装docker compose，但目前我安装的最新版DockerToolbox-1.11.2已经帮我们预装好了！So，你不需要折腾了！


### 端口映射和文件挂载

这两件事儿一般都是在你成功安装docker，并成功拥有了一个镜像后会碰到，一般你会运行这样的命令来启动一个容器：

```
docker run --name webserver -v /myweb:/etc/nginx/nginx.conf:ro -d -p 80:80 -d nginx
```

然后你打开DockerToolbox提供的Kitematic，选择virtualbox，你会看到确实已经运行了一个叫"webserver"的容器，切换到"Settings"界面的Port选项卡，你会看到并没有正确按照命令配置端口（或者一切正常~），不过没关系，可以直接在这个面板里进行配置。

这里需要注意，你只是将容器内部的指定端口映射到了virtualbox中docker虚拟机的指定端口，而非win下的指定端口，所以你直接在win下浏览器上使用"localhost"是无法访问的！这里有一篇[文章](http://www.cnblogs.com/bjfuouyang/p/3798421.html)教你如何配置virtualbox的网络端口映射来允许我们使用"localhost"。

接下来我们切换到"Settings"界面的Volumes选项卡，同样会发现并没有按照我们的命令成功映射本地文件夹到容器内！在这个界面上直接点击"change"按钮会给我们提示一些线索，原来是因为我们只能使用 **本地用户文件夹的内容** 作为映射地址，所以上面的命令应该改成：

```
docker run --name webserver -v /c/Users/kazaff/myweb:/etc/nginx/nginx.conf:ro -d -p 80:80 -d nginx
```

注意，这里我使用的"/c/Users/kazaff"前缀是根据我本地真实路径写的，你可能需要查看一下自己的对应路径，在终端中执行:

```
pwd ~
```

即可查看你应该使用的路径前缀。

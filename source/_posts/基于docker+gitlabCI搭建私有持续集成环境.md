title: 基于docker+gitlabCI搭建私有集成环境
date: 2016-06-15 09:37:12
tags:
- gitlab
- gitlab-ci
- gitlab-runner
- 持续集成
- ubuntu
- docker
categories: 运维
---

看了几天的docker，感觉好极了。现在回到我一开始的目标：构建一个团队内部的持续集成（下文统称CI）环境，并梳理出适合我们自己的工作流。

今天我们主要是来搭建依赖的环境：

- virtualBox
- ubuntu server 16
- docker
- gitlab version8+(该版本以上自带CI模块)
- gitlab runner
- gitlab需要的其它组件（redis，postgresql）

之前学习docker时，一直都是基于自己的工作机，装的是win7 64bit，win下和docker相关的问题可以看我之前的[文章](http://blog.kazaff.me/2016/06/13/docker%20for%20win/)。由于我们现在是要搭建一个自动化CI环境，提供web ui来供用户使用，所以环境搭建部分的操作完全不需要使用者参与，就不存在之前考虑的所谓的OS水土不服问题！介于网上多数资料都是使用ubuntu来作为docker的宿主系统，所以我这里就放弃了亲爱的centos（其实centos也是没问题的，我只是低调的炫耀自己通吃而已~见谅）。

老规矩，先来共享一下相关的文献资料：

- [Gitlab CI 文档](https://doc.gitlab.cc/ce/ci/)
- [使用Gitlab CI & Docker搭建CI环境](http://walterinsh.github.io/2016/04/18/using-gitlab-ci.html)(主要借鉴该文章后半部分提到的“提升构建速度”)
- [sameersbn/gitlab](https://hub.docker.com/r/sameersbn/gitlab/)(gitlab docker镜像的使用文档)


### ubuntu下安装docker

我们这次不使用win下的dockerToolbox来安装docker了，而是直接在虚拟机中安装unbuntu系统，然后直接在unbuntu中安装docker：

```
curl -sSL https://get.docker.com/ | sudo sh
```

目前这样的安装方式，依然需要注意，后面运行gitlab容器时配置的端口映射只是将容器的端口映射到了ubuntu虚拟机系统上，如果想让物理机直接访问，还需要在virtualBox上再进行一次端口映射，具体步骤在[这篇文章](http://blog.kazaff.me/2016/06/13/docker%20for%20win/)中我已经详细介绍过了。

### 镜像选择问题

docker环境装好后，就需要开始下载所需要的镜像了：

- sameersbn/gitlab:latest
- sameersbn/postgresql:latest
- sameersbn/redis:latest
- sameersbn/gitlab-ci-multi-runner:latest

由于我们是在国内，所以直接使用hub.docker.com来下载无异于浪费生命，还好国内也有良心镜像库：

- [sameersbn/gitlab:latest](https://hub.alauda.cn/repos/sameersbn/gitlab)
- [sameersbn/postgresql:latest](https://hub.alauda.cn/repos/sameersbn/postgresql)
- [sameersbn/redis:latest](https://hub.alauda.cn/repos/sameersbn/redis)
- [sameersbn/gitlab-ci-multi-runner:latest](https://hub.alauda.cn/repos/sameersbn/gitlab-ci-multi-runner)

这里需要提醒的是，尽管灵雀云上显示的`sameersbn/gitlab`镜像版本是"7.14.1"，不过别担心，只是md文件没有更新而已，可以切换到页面的“版本”选项卡来确认`latest`版本。

这样基本上20分钟就可以把三个镜像全部下载到本地啦~


### 启动容器

镜像下载完毕后，根据官方文档我们需要创建对应的docker容器来启动相关的服务，我并没有使用官方提供的第一种方法（docker-compose），主要是不熟悉~

我们手动来完成三个容器的启动：

```
#启动postgresql容器
docker run --name gitlab-postgresql -d \
    --env 'DB_NAME=gitlabhq_production' \
    --env 'DB_USER=gitlab' --env 'DB_PASS=password' \
    --env 'DB_EXTENSION=pg_trgm' \
    --volume /srv/docker/gitlab/postgresql:/var/lib/postgresql \
    index.alauda.cn/sameersbn/postgresql

#启动redis容器
docker run --name gitlab-redis -d \
    --volume /srv/docker/gitlab/redis:/var/lib/redis \
    index.alauda.cn/sameersbn/redis

#最后启动gitlab容器
docker run --name gitlab -d \
    --link gitlab-postgresql:postgresql --link gitlab-redis:redisio \
    --publish 10022:22 --publish 10080:80 \
    --env 'GITLAB_PORT=10080' --env 'GITLAB_SSH_PORT=10022' \
    --env 'GITLAB_SECRETS_DB_KEY_BASE=kazaffisagoodcoder' \
    --volume /srv/docker/gitlab/gitlab:/home/git/data \
    index.alauda.cn/sameersbn/gitlab
```

注意，上面的命令我已经改成使用我们本地镜像的名字了，还有在启动gitlab容器时我使用了一个自定义的字符串来赋值`GITLAB_SECRETS_DB_KEY_BASE`，由于是本地测试环境，外加我实在不知道如何通过在`--env`命令中使用shell变量来使用`pwgen -Bsv1 64`生成的随机字符串（手动输入简直等于自杀！）~知道如何做的朋友请留言赐教，不胜感激。

根据自己的硬件配置，可能需要稍等那么一分钟左右，在做好物理机端口映射配置后，我们就可以在本地浏览器中访问[http://localhost:10080/](http://localhost:10080/)来使用gitlab了！（可能由于gitlab首次启动初始化问题，你可能会看到gitlab提示的500错误，稍等一下再刷新即可）

现在你基本上就已经拥有了本地的gitlab环境，你可以使用本地的git客户端来创建一个gitlab测试项目，试试clone到本地，试试commit到gitlab，应该妥妥的~


### Gitlab Runner

这部分内容文章开头给的文档并没有汉化，不过没关系，我们都懂英文，对吧？

出于性能考虑官方不推荐将Runner安装在和gitlab同一台物理机上，出于安全考虑也不推荐将其安装在独立的物理机上，靠，也只能用docker来跑了！

这里还有两个概念：

- 特定Runner
- 共享Runner

前者适配有特殊需求的项目，同时也可以避免重要的项目runner资源被占用的问题。

而如果多个项目的优先级一样，并且有非常相似的依赖环境，那使用共享Runner是一个不错的选择。

既然我们决定使用docker容器来跑runner，那就先选个镜像吧，官方提供的是：[gitlab-ci-multi-runner](https://hub.docker.com/r/sameersbn/gitlab-ci-multi-runner/)，我是用的[灵雀云镜像](https://hub.alauda.cn/repos/sameersbn/gitlab-ci-multi-runner)了，注意，这里灵雀云镜像上的md介绍依然是错误的，镜像本身是和官方镜像一致的，小不完美啊 :———(~

还要说的一点是，官方提供的这个镜像是纯净镜像，不包含任何其它的环境（例如java环境或node环境），而你需要根据自己的项目实际情况来创建满足的自定义镜像。官方提供了几个语言环境的[例子](https://doc.gitlab.cc/ce/ci/examples/README.html)~

这种使用runner的思路相当于我们拿这个运行着runner的docker容器当作一个“物理机”，然后在这台“物理机”上做`.gitlab-ci.yml`指定要做的事儿。在此基础上，我们依然可以让Runner执行docker build模式，这里就涉及到“docker-in-docker”的思想了。具体细节官方提供了比较清晰的[文档](https://doc.gitlab.cc/ce/ci/docker/using_docker_build.html)。（其实，如果你使用文档中提到的“docker-in-docker”方案的话，那实际上就是“docker-in-(docker-in-docker)”，好绕啊~）

提到Runner的executor的类型，[文档](https://gitlab.com/gitlab-org/gitlab-ci-multi-runner/blob/master/docs/configuration/advanced-configuration.md#using-a-private-docker-registry)里有提到不少（shell，docker，ssh，等等），差别我感觉都不是太大，主要还是shell和docker两大类。

理论姿势就这么多，接下来我们跑起我的runner容器：

```
mkdir -p /srv/docker/gitlab-runner
docker run --name gitlab-ci-multi-runner -d --restart=always \
	--volume /srv/docker/gitlab-runner:/home/gitlab_ci_multi_runner/data \
	--link gitlab:gitlab --env='CI_SERVER_URL=http://gitlab/ci' \
	--env='RUNNER_TOKEN=你gitlab生成的token' \
	--env='RUNNER_DESCRIPTION=myrunner' --env='RUNNER_EXECUTOR=shell' \
	index.alauda.cn/sameersbn/gitlab-ci-multi-runner
```
这里需要提醒的是两点：

1. 你自己的gitlab生成的token可以在[http://localhost:10080/admin/runners](http://localhost:10080/admin/runners)看到（我假设你和我一样要创建的是一个共享Runner）
2. CI_SERVER_URL参数不要使用你物理机浏览器地址栏里的值，我们在上面的命令中已经将之前创建的gitlab容器link到了runner容器，so，我们只需要填写设置的主机名即可，否则无法注册成功！

接下来刷新你的gitlab管理员后台，你将会看到注册成功的Runner信息。

还没完，由于我们使用的是`shell`模式，所以我们还需要进入到runner容器中来安装docker环境：

```
docker exec -it gitlab-ci-multi-runner bash
curl -sSL https://get.docker.com/ | sudo sh
sudo usermod -aG docker gitlab-runner
sudo -u gitlab-runner -H docker info
```
如果可以看到正确的docker版本信息，那就说明一切顺利。**但怎么可能那么顺利！！** 事实证明我太天真了，在docker容器中由于文件系统是只读的，所以无法安装docker环境~至少我们现在的做法还不行，如下图：
![](http://pic.yupoo.com/kazaff/FCMGzWlk/143Y2S.png)

这里说一个插曲，当得到这个结论后的我失望的打算将这个悲剧的镜像删除，我在gitlab admin后台的runner页面点击“remove”按钮，然后又将对应的docker容器也删除掉。当我再次使用相同的命令打算启动一个干净的runner容器时，我发现gitlab的runner页面不在提示成功注册runner了！我彻底方了！一切都和第一次执行流程一致，为啥这次就不认了呢？

于是我发现了一个秘密，上面的创建runner容器的命令中有一个参数：
```
--volume /srv/docker/gitlab-runner:/home/gitlab_ci_multi_runner/data
```
我在宿主机的`/srv/docker/gitlab-runner`目录下找到了答案，原来一直试图想找到的`config.toml`文件在这里，也是因为这个文件的缘故所以gitlab才不在接受我的runner注册的！删除并重建这个目录即可！

走了弯路不可怕，可怕的是接下来不知道怎么办？没关系，我们现在先在宿主机上[安装gitlab-ci-multi-runner环境](https://gitlab.com/gitlab-org/gitlab-ci-multi-runner/blob/master/docs/install/linux-repository.md)，并使用官方提供的docker-in-docker方式来增加一台runner：

```
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-ci-multi-runner/script.deb.sh | sudo bash
sudo apt-get install gitlab-ci-multi-runner
sudo gitlab-ci-multi-runner register -n \
  --url http://localhost:10080/ci \
  --token 你gitlab生成的token \
  --executor docker \
  --description "My Docker Runner" \
  --docker-image "index.alauda.cn/library/docker:latest" \
  --docker-privileged
```
记住，我们这里要填写的gitlab-ci coordinator URL应该是：`http://localhost:10080/ci`。

这里还有个细节是官方文档没有提到的，那就是[让注册好的Runner运行起来](http://www.jianshu.com/p/2b43151fb92e)，不过其实默认安装好gitlab-ci-multi-runner后它就自己已经以服务的方式注册到系统里了（服务名：gitlab-runner），每次注册runner导致`config.toml`文件变更后都会自动触发该服务的reload。但了解一下这个细节还是对分析问题很有帮助的！

这样我们就应该有两个runner了（一个是shell类型，一个是docker类型）：
![](http://pic.yupoo.com/kazaff/FCT2FpnG/fzJZp.png)

//todo ubuntu -> docker-in-docker -> runner -> docker engine

### .gitlab-ci.yml

这个文件应该放在我们的项目repo的根目录下，用来描述当发生目标行为后，runner要做的工作。官方文档有对其语法的[描述](https://doc.gitlab.cc/ce/ci/yaml/README.html)。

环境基本上就搭建好了，下一篇我们就开始设计工作流了。

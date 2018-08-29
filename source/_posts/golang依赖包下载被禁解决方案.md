title: golang依赖包下载被禁解决方案
date: 2018-8-27 10:37:12
tags:
- golang
- 墙

categories:
- golang
---

每次装环境，都会卡在这里半天，实在不知道曾经的golang.org到底做错了什么而被封杀，有谁知道可以留言一下吗？

查了几个方案，一开始一心想按照前辈的经验搭建一个代理来下载目标库文件，不过后来发现实在是太麻烦了就放弃了。

今天我说一个土鳖但直观的方案，那就是手动下载项目依赖包文件：

1. 确保系统上安装了git
2. `git clone https://github.com/golang/xxx`到本地指定目录（其中xxx表示你要安装的某个依赖库）
3. 在`$GOPATH/src/golang.org/x/`下手动创建对应的文件夹结构
4. 将步骤2中下载来的目标文件目录拷贝到步骤3手动创建的目录中
5. 重新执行`go get xxxxx`来完成安装

就是这么简单，或者更简单的是，你自己本地搭建一个web环境，然后把golang.org配置在hosts里指向自己的web环境即可。

 举一个实际的例子吧，假如我们要安装`github.com/go-resty/resty`，直接执行`go get`你会得到下面的错误：
 
 ```
package golang.org/x/net/publicsuffix: unrecognized import path "golang.org/x/net/publicsuffix" (https fetch: Get https://golang.org/x/net/publicsuffix?go-get=1: dial tcp 216.239.37.1:443: i/o timeout)

 ```

那怎么办？我们按照上面的步骤，先`git clone https://github.com/golang/net`到本地。然后在本地的`GOPATH`目录下创建`/src/golang.org/x/net/`目录纵深。
然后直接把`git clone`的目录中的`publicsuffix`文件夹拷贝到手动创建的目录中即可，最后再次执行`go get github.com/go-resty/resty`即可。
title: bitcoin开发环境搭建
date: 2018-04-24 09:37:12
tags:
- golang
- btcd
- docker

categories:
- blockchain
---

区块链，哼，我倒要看看能有多火，能有多难！

我们这次就来搭建一下bitcoin的开发环境，作为一个新手，肯定少不了要了解非常多的基本概念和常识，才能顺利的完成开发环境的搭建，所以这篇文章不仅仅是搭建一个简单的开发环境而已，也会争取尽可能多的记录一些基础知识点。这样，我觉得才算是有价值。

bitcoin的钱包客户端不止一种，而我比较中意的是golang的[btcd](https://github.com/btcsuite/btcd)。据说基于golang的性能出众，目前很多矿机上都是跑的golang客户端，不确定是否属实，也可能人家都进行了各自的改良，但我目前就打算以btcd作为入口，希望你也和我的选择一样。

### 项目介绍

在github上btcd主项目下根据功能和层次，拆分了好多不同的子项目，我们首先就来了解一下它们：

- blockchain: 这个模块应该是核心模块，负责区块链基础功能（创建，校验，存储区块），不过目前对我们这种小白来说，可以略过
- btcec: 提供加解密和数字签名相关库
- btcjson: 用于针对bitcoin JSON-RPC API规范进行参数编码解码的库
- chaincfg: 提供了预设的bitcoin网络配置项
- cmd: 提供了一些命令工具
- connmgr: 负责bitcoin网络链接管理（链接的生命周期管理）
- database: 存储区块数据和一些元数据
- mempool: 暂存和管理待处理交易信息的模块
- mining: 应该是挖矿相关功能模块
- netsync: 提供一个并发安全的同步锁协议
- peer: 创建和管理网络节点的模块
- rpcclient: 实现了一个基于websocket的json-rpc客户端
- wire: 提供了bitcoin的通信协议实现

上面每个模块的描述是从项目文件夹中对应的`README.md`中翻译的，由于目前我也处于入门阶段，可能理解错了一些含义，希望大家可以帮我纠错~

值得关注的一点，是来自官方的一段描述：

> btcd和bitcoin core客户端功能上最大的不同点是btcd并不包含钱包相关功能，这是有意而为之的一个设计决策。这意味着，只安装btcd并不能进行交易，还需要安装运行btcwallet才行。

这段描述对我们搭建环境很重要，下面咱们就回到主题，搭建环境喽~

### 环境搭建

根据我的个人习惯，开发环境还是用docker来搭建好，你不需要安装go语言环境，不需要自己下载btcd编译环境等等，只需要简单的找到一个满足自己的镜像就可以了。
我根据自己找到的镜像，修改后的dockerfile共享出来：

```
FROM golang:1.9-stretch

MAINTAINER edisondik@gmail.com

# Expose mainnet ports (server, rpc)
EXPOSE 8333 8334

# Expose testnet ports (server, rpc)
EXPOSE 18333 18334

# Expose simnet ports (server, rpc)
EXPOSE 18555 18556

# Expose segnet ports (server, rpc)
EXPOSE 28901 28902

RUN go env GOROOT GOPATH

RUN go get -u github.com/Masterminds/glide \
&& git clone https://github.com/btcsuite/btcd $GOPATH/src/github.com/btcsuite/btcd \
&& cd $GOPATH/src/github.com/btcsuite/btcd \
&& glide install \
&& go install . ./cmd/... \
&& btcd --version \
&& cd $GOPATH/src/github.com/btcsuite/btcd \
&& git pull && glide install \
&& go install . ./cmd/... \
&& btcctl --version

RUN go get -u github.com/Masterminds/glide \
&&  git clone https://github.com/btcsuite/btcwallet $GOPATH/src/github.com/btcsuite/btcwallet \
&&  cd $GOPATH/src/github.com/btcsuite/btcwallet \
&&  glide install \
&&  go install . ./cmd/... \
&&  cd $GOPATH/src/github.com/btcsuite/btcwallet \
&&  git pull && glide install \
&&  go install . ./cmd/...


ENTRYPOINT ["/bin/sh"]
```

然后根据这个配置我们来创建一个容器，终端切到文件所在的文件夹下后执行：`docker build -t bctd-test .`

进入到我们刚创建的容器中，执行：

```
btcd --simnet --rpcuser=kazaff --rpcpass=123456
btcwallet --simnet --username=kazaff --password=123456
```

如果碰到错误提醒：

```
[WRN] BTCW: open /root/.btcwallet/btcwallet.conf: no such file or directory
or
Error creating a default config file: open /root/.btcwallet/btcwallet.conf: no such file or directory
```

不要怕，只需要在对应目录下手动创建对应文件即可。官方说明中提示，需要在对应的配置文件中定义好rpc和rpclimited对应帐号，btcd才会开始rpc服务，但我们通过上面的命令启动btcd后，就已经可以进行rpc请求了。

接下来我们来执行查询命令：

```
 btcctl --simnet --wallet --rpcuser=kazaff --rpcpass=123456 getinfo
```

注意，我们增加的`--wallet`，目前我理解的是，这个参数指明我们是去连接的wallet服务，wallet服务也会去调用btcd服务。上面的这个命令其实也可以去掉该参数，只是你得到的返回结果不太一样。但，个别命令只有wallet服务提供，所以还是都加上这个参数的稳妥一些。

是的，没错，用docker来搭建开发环境就是这么简单，基本上我们已经搭建好了一个模型。接下来我们来逐个测试命令。

### 命令介绍

```
 btcctl --simnet --wallet --rpcuser=kazaff --rpcpass=123456 listaccounts
```

该命令会列出目前btcd中包含的帐号列表。

我们还可以查询指定帐号下的钱包地址列表：

```
 btcctl --simnet --wallet --rpcuser=kazaff --rpcpass=123456 getaddressesbyaccount "default"
```

该命令查看了默认帐号下所有的钱包地址。

那到底有多少可以互动的命令呢？

```
 btcctl --simnet --wallet --rpcuser=kazaff --rpcpass=123456 help
```
超级多~~我找到一个不确定是否过期的[文档](https://www.51chain.net/portal/book/btcapi/3.html)，可以快速对可用命令了解一个大概~~

后面我还会分享更多的关于用代码和钱包服务进行交互的内容，长期关注吧~

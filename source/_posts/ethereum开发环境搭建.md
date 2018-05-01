title: ethereum开发环境搭建
date: 2018-04-30 09:37:12
tags:
- golang
- geth
- docker

categories:
- blockchain
---

区块链，哼，我倒要看看能有多火，能有多难！

本次我们来搭建一个以太坊的私有链，大体思路和前一篇Bitcoin的也很类似。

早先其实就在笔记本上搭建过一个环境，不过玩了几次就荒废了。这次记录下来，以备不时之需~~我们还是借助docker来快速搭建一套开发环境，所以第一步还是选择一个方便一点的镜像。

我找了好几个，都感觉不是太顺手，所以在别人的基础上修改了一下：

```
FROM ubuntu:latest

LABEL maintainer="edisondik@gmail.com"

ARG GETH_URL=https://gethstore.blob.core.windows.net/builds/geth-alltools-linux-amd64-1.7.2-1db4ecdc.tar.gz
ARG GETH_MD5=c17c164d2d59d3972a2e6ecf922d2093
ARG DEBIAN_FRONTEND=noninteractive

RUN apt update && \
    apt install wget -y && \
    cd /tmp && \
    wget "$GETH_URL" -q -O /tmp/geth-alltools-linux-amd64.tar.gz && \
    echo "$GETH_MD5  geth-alltools-linux-amd64.tar.gz" > /tmp/geth-alltools-linux-amd64.tar.gz.md5 && \
    md5sum -c /tmp/geth-alltools-linux-amd64.tar.gz.md5 && \
    tar -xzf /tmp/geth-alltools-linux-amd64.tar.gz -C /usr/local/bin/ --strip-components=1

ENV GEN_NONCE="0xeddeadbabeeddead" \
    DATA_DIR="/root/.ethereum" \
    CHAIN_TYPE="private" \
    GEN_CHAIN_ID=70213

WORKDIR /opt

# like ethereum/client-go
EXPOSE 30303
EXPOSE 8545

# bootnode port
EXPOSE 30301
EXPOSE 30301/udp

ADD src/* /opt/
RUN chmod +x /opt/*.sh

# ENTRYPOINT ["/opt/startgeth.sh"]
ENTRYPOINT ["/bin/sh"]

```

上面这个dockerfile文件的同级目录下需要`src`文件夹，在改文件夹下放2个文件:

genesis.json:
```
{
  "config": {
    "chainId": ${GEN_CHAIN_ID},
    "homesteadBlock": 0,
    "eip155Block": 0,
    "eip158Block": 0
  },
    "alloc"      : {
    ${GEN_ALLOC}
  },
  "coinbase"   : "0x0000000000000000000000000000000000000000",
  "difficulty" : "0x20000",
  "extraData"  : "",
  "gasLimit"   : "0x2fefd8",
  "nonce"      : "${GEN_NONCE}",
  "mixhash"    : "0x0000000000000000000000000000000000000000000000000000000000000000",
  "parentHash" : "0x0000000000000000000000000000000000000000000000000000000000000000",
  "timestamp"  : "0x00"
}
```

startgeth.sh:
```
#!/bin/sh

GEN_ARGS=

# replace vars
echo "Generating genesis.nonce from arguments..."
sed "s/\${GEN_NONCE}/$GEN_NONCE/g" -i /opt/genesis.json

echo "Generating genesis.alloc from arguments..."
sed "s/\${GEN_ALLOC}/$GEN_ALLOC/g" -i /opt/genesis.json

echo "Generating genesis.chainid from arguments..."
sed "s/\${GEN_CHAIN_ID}/$GEN_CHAIN_ID/g" -i /opt/genesis.json

echo "Running ethereum node with CHAIN_TYPE=$CHAIN_TYPE"

# empty datadir -> geth init
DATA_DIR=${DATA_DIR:-"/root/.ethereum"}
echo "DATA_DIR=$DATA_DIR, contents:"
ls -la $DATA_DIR

echo "DATA_DIR '$DATA_DIR' non existant or empty. Initializing DATA_DIR..."
geth --datadir "$DATA_DIR" init /opt/genesis.json

GEN_ARGS="--datadir $DATA_DIR"
#  [[ ! -z $NET_ID ]] && GEN_ARGS="$GEN_ARGS --networkid=$NET_ID"
#  [[ ! -z $MY_IP ]] && GEN_ARGS="$GEN_ARGS --nat=extip:$MY_IP"
GEN_ARGS="$GEN_ARGS --nat=any"


echo "Running geth with arguments $GEN_ARGS"
exec /usr/local/bin/geth --nodiscover --rpc --rpcapi db,eth,net,web3,personal $GEN_ARGS

```

然后创建这个镜像即可（`docker build -t kz-geth .`）。

等待片刻（其实很久），我们接下来就可以执行`./startgeth.sh`命令来启动以太坊节点了。不过，稍微解释一下dockerfile里的细节，“GEN_CHAIN_ID”一定要写一个自己的值，如果默认用1的话，启动的以太坊节点会去同步主链上的数据，很恐怖的哟~

启动完毕后，我们可以再开启一个新的终端，在其中执行`geth attach`命令，它会自动开启一个console并链接到我们自己的节点上。在这个console界面，我们就可以执行一些常见的命令来和以太坊节点交互了。

下面我列一下简单的命令来完成一次转账流程：

1. eth.accounts  该命令列出当前帐号有哪些，目前应该是空
2. personal.newAccount("123")  创建一个密码为123的帐号，需要执行两次，第一次生成的那个帐号默认会作为coinbase来接受挖矿奖励
3. miner.start() 开启挖矿，这样我们第一个帐号就会不停的得到以太币
4. eth.getBalance("第一个帐号地址") 可以查看指定帐号的余额
5. eth.sendTransaction({from:"第一个帐号地址", to: "第二个帐号地址", value: web3.toWei(1)}) 这样就能完成一次转账

当然，目前为止，我们只是完成了最最简单的一步，后面还有更多好玩的内容，期待~

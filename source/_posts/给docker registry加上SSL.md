title: 给docker registry加上SSL
date: 2016-06-20 09:37:12
tags:
- docker
- registry
- SSL
- shipyard
categories: 运维
---

之前我们的registry是裸奔的，今天我们就给它穿点衣服~
参考主流的做法，就是搭建一个nginx节点来做registry的反向代理，然后在nginx上配置ssl证书来达到安全校验的目的。
<!--more-->
基于docker的话，我们只需根据nginx的官方镜像创建一个容器即可：
```
docker run --name nginx --restart=always -p 443:443 -v /www:/www \
-v /www/nginx/nginx.conf:/etc/nginx/nginx.conf:ro -d --link registry:registry \
nginx:latest
```
先别着急执行上面这条命令，首先，我们注意两点：

- link了registry容器，该容器就是我们的裸奔registry
- 挂在了www文件夹，这里面就放了SSL使用的证书

跟随[这篇文章](http://blog.coocla.org/docker-private-registry.html)的生成证书流程，完成了所需要的证书文件后，我们来配置一下nginx的配置文件：
```
...
http {
        sendfile on;
        keepalive_timeout 65;

        upstream registry {
                server registry:5000;
        }

        server {
                listen 443;
                server_name registry.me;

                ssl on;
                ssl_certificate /www/nginx/private/registry.me.crt;
                ssl_certificate_key /www/nginx/private/registry.me.key;

                client_max_body_size 0;
                chunked_transfer_encoding on;

                #location /v2/ {
                #
                #}
                location / {
                        proxy_pass      http://registry;
                        proxy_set_header        HOST    $http_host;
                        proxy_set_header        X-Real-IP       $remote_addr;
                        proxy_set_header        X-Forwarded-For $proxy_add_x_forwarded_for;
                        proxy_set_header        X-Forwarded-Proto       $scheme;
                        proxy_read_timeout      900;
                }
        }
}
```
我们这里给nginx配置的`server_name`和生成证书时输入的`Common Name`保持一致。这样就可以run了。

然后我们还需要给物理机映射对应的443端口，另外还有修改hosts文件，增加域名绑定。之后就可以试试了：
```
curl -vk https://registry.me/v2/_catalog
```
注意，由于我们的registry使用的是v2版本~

### shipyard

命令行下的docker使用，总不是那么省心，所以我们可以安装[shipyard](http://shipyard-project.com/docs/deploy/automated/)，它提供了web ui来帮助我们更好的使用docker。

注意，官方提供的`deploy`脚本中使用的都是dockerhub上的镜像，我们国内下载起来非常慢，外加sh脚本执行时进度提醒不及时，很容易白等一整天，推荐的方式是自己先将所需的镜像pull到本地，再执行`deploy`脚本。这里不推荐切换`deploy`中的镜像为国内镜像源，主流的几个国内镜像仓库都没有匹配的源，会导致版本不兼容的~

安装好以后会暴露8080端口和一个admin帐号（密码是shipyard），你可以浏览器上直接访问了（记得做物理机端口映射）。

不幸的是，目前官方主版本并不支持registry的v2版本接口，所以我们无法在shipyard中添加私有仓库。虽然我在github上看到有支持v2的branch提供我们使用，但并没有提供足够详细的文档，我这个新手不知道如何部署啊~可悲~如果你知道，请留言赐教~不胜感激！

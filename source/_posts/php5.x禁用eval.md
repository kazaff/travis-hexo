title: php5.x禁用eval
date: 2018-10-19 09:37:12
tags:
- php
- eval
- linux

categories:
- 运维
---

如果是我的死忠粉，你应该看得出来，这几天我在修补公司服务器的安全漏洞。
我并不是专家，干的也都只是苦力活，但也要记录一下，人老了，总忘。

这次我们来说如何禁止php代码中执行eval函数，本来以为直接修改php.ini中的disable_function即可~
但现实往往并不是那么如意，查了一下GG，发现原来eval并非函数，而是php底层提供的一种特性。

幸好有前辈提供了php扩展来禁用万恶的eval：[suhosin](https://www.suhosin.org/stories/download.html)
一开始发现是需要给php打补丁，我是拒绝的，但确实没有找到更好的方法。不过实际安装下来，真的很方便：

```
yum install wget  make gcc gcc-c++ zlib-devel openssl openssl-devel pcre-devel kernel keyutils  patch perl

cd /usr/local/src

wget http://download.suhosin.org/suhosin-对应的版本.tgz

tar zxvf suhosin-对应的版本.tgz

cd suhosin-对应的版本

/usr/bin/phpize

./configure  --with-php-config=/usr/bin/php-config

make & make install
```

编译完后会提示你库文件的位置，例如: `/usr/lib64/php/modules`

我们只需要在php.ini中增加对应的扩展即可：
```
extension=/usr/lib64/php/modules/suhosin.so
suhosin.executor.disable_eval=On
```

重启php-fpm进程后，就可以在phpinfo中看到suhosin扩展已经装好了~
仔细看增加的配置项，其实很多控制的点，得慢慢研究啊~

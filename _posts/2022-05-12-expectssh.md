---
layout: post
title:  "Expect SSH"
date:   "2022-05-12 10:00:00"
---

提到自动登录ssh的时候，最好还是使用秘钥登录，当然也可以使用expect。

下面介绍几个expect和ssh的例子。

### login

-p是指定远程的端口。

interact执行完成后保持交互状态，把控制权交给控制台，这个时候就可以手工操作了。

```

spawn ssh -o ServerAliveInterval=60 -p 11111 username@ip
expect "*password:"
send "Password\r"
interact

```

### proxy

当需要使用ssh做socket代理的写法，在本地开一个10001的端口。

```

spawn ssh -N -g -o ServerAliveInterval=60 -D 10001 -p 11111 username@ip
expect "*password:"
send "Password\r"
interact

```


### login by proxy

通过一个本地的代理端口10000来登录ssh。

nc的-X 5是指定Socks V5，-x 是指定代理地址。

```

spawn ssh -o ServerAliveInterval=60 -o "ProxyCommand=nc -X 5 -x 127.0.0.1:10000 %h %p" -p 11111 username@ip
expect "*password:"
send "Password\r"
interact

```


### proxy by proxy

将上面的命令组合一下就好了。

```

spawn ssh -N -g -o ServerAliveInterval=60 -o "ProxyCommand=nc -X 5 -x 127.0.0.1:10000 %h %p" -D 10001 -p 11111 username@ip
expect "*password:"
send "Password\r"
interact

```

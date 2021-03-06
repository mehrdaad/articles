# SSH 端口转发

## 简介

SSH 除了登录服务器，还有一大用途，就是作为加密通信的中介，充当两台服务器之间的通信加密跳板，使得原本不加密的通信变成加密通信。这个功能称为端口转发（port forwarding），又称 SSH 隧道（tunnel）。

端口转发的两个主要作用：（1）将不安全的网络服务变成 SSH 安全连接，比如通过端口转发访问 Telnet、FTP 等明文服务；（2）作为数据通信的加密跳板，绕过网络防火墙。

端口转发有三种使用方法：动态转发，本地转发，远程转发。

## 动态转发

动态转发指的是，本机与 SSH 服务器之间创建了一个加密连接，然后本机内部针对某个端口的通信，都通过这个加密连接转发。它的一个使用场景就是，访问所有外部网站，都通过 SSH 转发。

动态转发需要把本地端口绑定到 SSH 跳板机。至于 SSH 跳板机要去访问哪一个网站，完全是动态的，取决于原始通信，所以叫做动态转发。

```bash
$ ssh -D local-port tunnel-host -N
```

上面命令中，`-D`表示动态转发，`local-port`是本地端口，`tunnel-host`是 ssh 跳板机，`-N`表示只进行端口转发，不登录远程 Shell。

举例来说，如果本地端口是`2121`，那么实际命令就是下面这样。

```bash
$ ssh -D 2121 tunnel-host -N
```

注意，这种转发采用了 SOCKS5 协议。访问外部网站时，需要把 HTTP 请求转成 SOCKS5 协议，才能把本地端口的请求转发出去。

下面是 ssh 隧道建立后的一个使用实例。

```bash
$ curl -x socks5://localhost:2121 http://www.example.com
```

上面命令中，curl 的`-x`参数指定代理服务器，即通过 SOCKS5 协议的本地`2121`端口，访问`http://www.example.com`。

## 本地转发

本地转发（local forwarding）指的是，SSH 服务器作为跳板机，建立本地计算机与特定目标网站之间的加密连接。

```html
$ ssh -L local-port:target-host:target-port tunnel-host
```

上面命令中，`-L`参数表示本地转发，`local-port`是本地端口，`target-host`是你想要访问的目标服务器，`target-port`是目标服务器的端口，`tunnel-host`是 SSH 跳板机。

举例来说，现在有一台 SSH 跳板机`tunnel-host`，我们想要通过这台机器，在本地`2121`端口与目标网站`www.example.com`的80端口之间建立 SSH 隧道，就可以写成下面这样。

```bash
$ ssh -L 2121:www.example.com:80 tunnel-host -N
```

然后，访问本机的`2121`端口，就是访问`www.example.com`。

```bash
$ curl http://localhost:2121
```

注意，本地端口转发采用 HTTP 协议，不用转成 SOCKS5 协议。

另一个例子是加密访问邮件获取协议 POP3。

```bash
$ ssh -L 1100:mail.example.com:110 mail.example.com
```

上面命令将本机的1100端口，绑定邮件服务器`mail.example.com`的110端口（POP3 协议的默认端口）。端口转发建立以后，POP3 邮件客户端只需要访问本机的1100端口，请求就会自动转发到`mail.example.com`的110端口。

上面这种情况有一个前提条件，就是`mail.example.com`必须运行 SSH 服务器。否则，就必须通过另一台服务器中介，执行的命令要改成下面这样。

```bash
$ ssh -L 1100:mail.example.com:110 other.example.com
```

上面命令中，本机的1100端口还是绑定`mail.example.com`的110端口，但是由于`mail.example.com`没有运行 SSH 服务器，所以必须通过`other.example.com`中介。本机的 POP3 请求通过1100端口，先发给`other.example.com`的22端口（sshd 默认端口），再由后者转给`mail.example.com`，得到数据以后再原路返回。

注意，采用上面的中介方式，只有本机到`other.example.com`的这一段是加密的，`other.example.com`到`mail.example.com`的这一段并不加密。

这个命令最好加上`-N`参数，表示不在 SSH 跳板机执行远程命令，让 SSH 只充当隧道。另外还有一个`-f`参数表示 SSH 连接在后台运行。

## 远程转发

远程端口转发与本地端口转发相反。它将连接从服务器主机转发到本地主机，然后再转发到目标主机端口。

这种场景比较特殊，主要针对内网的情况。本地计算机在外网，ssh 跳板机和目标服务器都在内网，而且本地计算机无法访问内网之中的跳板机，但是跳板机可以访问本机计算机。

由于本机无法访问跳板机，就无法从外网发起隧道，必须反过来，从跳板机发起隧道，这时就会用到远程端口转发。

```bash
$ ssh -R local-port:target-host:target-port -N local
```

上面的命令，首先需要注意，不是在本机执行的，而是在 ssh 跳板机执行的，从跳板机去连接本地计算机。`-R`参数表示远程端口转发，`local-port`是本地计算机的端口，`target-host`和`target-port`是目标服务器及其端口。

比如，跳板机执行下面的命令，绑定本地计算机的`2121`端口，去访问`www.example.com:80`。

```bash
$ ssh -R 2121:www.example.com:80 local -N
```

执行上面的命令以后，跳板机到本地计算机的隧道已经建立了。然后，就可以从本机访问目标服务器了，即在本机执行下面的命令。

```bash
$ curl http://localhost:2121
```

这种端口转发会远程绑定另一台机器的端口，所以叫做“远程端口转发”，也是采用 HTTP 协议。

## 实例：Email 加密下载

上面介绍了 ssh 端口转发的三种用法，下面就来看几个具体的实例。

公共场合的 WiFi，如果使用非加密的通信，是非常不安全的。假定我们在咖啡馆里面，需要从邮件服务器明文下载邮件，怎么办？一种解决方法就是，采用本地端口转发，在本地电脑与邮件服务器之间，建立 ssh 隧道。

```bash
$ ssh -L 2121:mail-server:143 tunnel-host -N
```

上面命令指定本地`2121`端口绑定 ssh 跳板机`tunnel-host`，跳板机连向邮件服务器的`143`端口。这样下载邮件，本地到 ssh 跳板机这一段，是完全加密的。当然，跳板机到邮件服务器的这一段，依然是明文的。

## 实例：简易 VPN

VPN 用来在外网与内网之间建立一条加密通道。内网的服务器不能从外网直接访问，必须通过一个跳板机，如果本机可以访问跳板机，就可以使用 ssh 本地转发，简单实现一个 VPN。

```bash
$ ssh -L 2080:corp-server:80 -L 2443:corp-server:443 tunnel-host -N
```

上面命令通过 ssh 跳板机，将本机的`2080`端口绑定内网服务器的`80`端口，本机的`2443`端口绑定内网服务器的`443`端口。

## 实例：两级跳板

端口转发可以有多级，比如新建两个 ssh 隧道，第一个隧道转发给第二个隧道，第二个隧道才能访问目标服务器。

首先，新建第一级隧道。

```bash
$ ssh -L 7999:localhost:2999 tunnel1-host
```

上面命令在本地`7999`端口与`tunnel1-host`之间建立一条隧道，隧道的出口是`tunnel1-host`的`localhost:2999`，也就是`tunnel1-host`收到本机的请求以后，转发给本机的`2999`端口。

然后，在第一台跳板机执行下面的命令，新建第二级隧道。

```bash
$ ssh -L 2999:target-host:7999 tunnel2-host -N
```

上面命令将第一台跳板机`tunnel1-host`的`2999`端口，通过第二台跳板机`tunnel2-host`，连接到目标服务器`target-host`的`7999`端口。
 
最终效果就是，访问本机的`2999`端口，就会转发到`target-host`的`2999`端口。

## 参考链接

- [An Illustrated Guide to SSH Tunnels](https://solitum.net/an-illustrated-guide-to-ssh-tunnels/), Scott Wiersdorf


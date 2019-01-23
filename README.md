# nginx-demo 

### nginx的学习笔记

### 参考

http://www.nginx.cn/doc/

程序员技术交流、面试、职场、生活、游戏、相亲，综合讨论QQ群：387017550，群内经常发红包，欢迎加入

### 0、命令行工具

由于需要涉及到比较多的命令行输入，所以建议搞一个专业的命令行工具，mac自带的终端功能并不强大。

<b>iTerm2</b>，参考：https://www.jianshu.com/p/33f2048b8862

---

### 目录

<a href='https://github.com/qq20004604/nginx-demo/blob/master/01、安装nginx.md'>01、安装nginx</a>

<a href='https://github.com/qq20004604/nginx-demo/blob/master/02~04、目录和命令.md'>02~04、目录和命令</a>

<a href='https://github.com/qq20004604/nginx-demo/blob/master/05、配置nginx的配置文件.md'>05、配置nginx的配置文件</a>

## 06、一个nginx服务器为多个ip服务

简单来说，访问多个ip，都会交由同一个nginx服务器处理，并且显示不同页面内容。

举例来说，通常，访问 192.168.0.151 这个ip会访问 A服务器，访问 192.168.0.152 这个ip会访问 B服务器。

假如你有2个服务，但只有一台机子。

* 在之前，你需要通过 a.com/a 来访问同一个服务器 A 服务；
* 访问 a.com/b 来访问同一个服务器的 B 服务；
* 这样才解耦方便独立管理；
* 缺点是 A、B 两个服务，还是有一定耦合度，也不方便。

<br/>
但假如你想将这2个拆分出来，像正常使用了2个服务器一样，就可以配置基于 ip 的虚拟主机，并利用 nginx 通过配置 server属性，来让其在访问不同 ip 的时候，访问同一台机子的不同静态文件（这是最基础的用法）。

举例来说，我遇见这样一个场景：

* 我有 A 页面、B 页面；
* 我希望访问 192.168.0.151 时，进入 A 页面，访问 192.168.0.152 时，进入 B 页面；
* 现在 A 页面比较简单，但以后 A 页面可能会复杂起来，可能会使用一个独立的服务器；
* 为了对用户友好（不需要自己改变保存的地址），因此希望用户一直访问同一个ip即可，对于我切换是无感知的；
* 虽然以后 A 会独立出来，但现在A、B分别用一个服务器就太浪费了；

所以我期望是这样的：

* 我现在只使用一个服务器；
* 访问 192.168.0.151 和 192.168.0.152 时，都会访问到这个服务器（多IP对单服务器）；
* 访问 192.168.0.151 时，进入 A 页面，访问 192.168.0.152 时，进入 B 页面；
* 以后 A 独立后，直接开一个新服务器，访问 192.168.0.151 时，去访问新服务器；

实现方法：

1. 先判断 192.168.0.151 和 192.168.0.152 是否有机器存在（方法：ping 两个ip）；
2. 添加 IP 别名，让以上2个IP，在访问的时候指向 nginx 所在服务器；
    1. 具体来说，ifconfig 找到网卡名、ip地址，netmask，broadcast；
    2. 输入以下命令（记得替换 [] 内的内容）；
    3. ``/sbin/ifconfig [网卡名]:1 [新的ip] broadcast [原本网卡的broadcast的值] netmask [原netmask值] up``
    4. ``/sbin/route add -host  [新的ip] dev  [网卡名]:1``
    5. 然后ifconfig查看是否添加成功；
    6. 如果添加多个，则网卡名后面的 :1 变为 :2 ，依次类推。也可以 :1 改为 :abc 之类的；
    7. 以上的临时添加方案（重启后失效）；
    8. 永久方案是，将以上内容，添加到 ``/etc/rc.local``文件的末尾，然后保存即可；
3. 在nginx的配置文件 ``nginx/conf/nginx.conf``　文件中，原本 server 属性的同级位置添加以下代码：
```
    # 第二个虚拟主机
    server {
        listen       192.168.0.151:80;
        server_name  192.168.0.151 default;
    
        location / {
            root   html;
            index  index.html index.htm;
        }
    
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
    # 第三个虚拟主机
    server {
        listen       192.168.0.152:80;
        server_name  192.168.0.152;
    
        location / {
            root   html2;
            index  index.html index.htm;
        }
    
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }
    }
```
4. 这代表如果访问 192.168.0.151 ，则会访问 ``./html`` 文件夹，默认是 index.html；
5. 访问 192.168.0.151 ，则会访问 ``./html2`` 文件夹中的文件，默认是 index.html；

测试方案：

1. 先按以上做；
2. 将原本 nginx 文件夹下的 html 文件夹，复制一份，改名 html2；
3. 更改 html2/index.html 的内容，例如删除正文；
4. 执行 ``nginx -s reload`` 重新读取配置文件（因为在上面更新过一次）；
5. 依次访问 192.168.0.151 和 192.168.0.152 查看是否正常；
6. 建议先尝试非永久性改变，正常后再去改配置文件实现永久性改动；


### xx、根目录

在配置 ``nginx.conf`` 文件时，经常要涉及到目录的配置。

在默认配置文件中，路径里写的都是相对路径，这个相对路径，其基础指的就是 nginx 的安装目录。

那么如何确定 nginx 的安装目录呢？

linux下可以使用这个命令：

```
ps -ef | grep nginx  
```

mac下不行，不会显示目录，但是更简单，直接在这个目录下找吧 ``/usr/local/Cellar/nginx/``


### xx、主进程pid

当我们需要控制 nginx 的主进程时，我们需要知道主进程的 pid

获取pid时，可以通过 nginx.pid 来获取，里面只有一个值，就是 主进程的 pid 的值；

一般他在 ``/usr/local/nginx/nginx.pid`` 中，如果你按照我的方式来安装的话，他应该在 ``/usr/local/var/run/nginx.pid`` 中。

如果需要迁移（例如 ``nginx.conf`` 里，这个文件应该在 ``nginx/logs`` 这个目录中。但在mac里，他并不是）

那么就在命令行中，先进入目标目录（如在 ``nginx/logs`` 目录下），输入以下命令移动文件：

```$xslt
mv /usr/local/var/run/nginx.pid ./
```

拿到主进程pid的意义在哪，待后续补充；

```
// todo 待补充
```

### 08、信号处理

这篇文章讲的比较好：

<a href="https://blog.csdn.net/zwan0518/article/details/49851273">nginx介绍-信号处理</a>


## Tips1 不使用管理员权限运行docker命令
docker命令默认情况下是需要使用root权限去运行，在不能使用管理员权限的场合下，可以通过创建一个名为docker的组，然后给这个组添加普通用户，
那么添加的普通用户就可以不需要使用root权限去执行docker命令了，docker执行命令的时候本质上就是去连接unix socket，然后通过unix socket给
docker daemon发送命令，因此只要对unix socket文件有权限就可以操作docker，docker正是利用了这点，给unix socket设置了属组为docker，组权限为rw，
通过下面命令可以查看:

```
ls -l /var/run/docker.sock  
srw-rw---- 1 root docker 0  3月 17 09:25 /var/run/docker.sock
```

## Tips2 如何可视化容器间的link关系，和镜像层之间的关系

[dockviz](https://github.com/justone/dockviz)是官方推荐的一个第三方工具，可以将容器之间的link关系和镜像层之间的关系输出为dot脚本然后可
以通过dot命令将dot脚本输出为图形。

注: docker images -t可以显示镜像层树状关系。不过这个在较新版本的docker中已经被废弃了

## Tips3 如何设置镜像的自动构建
Setp1:  登陆Docker hub 创建自己的账号和密码，登陆后点击用户名选择[Settings](https://hub.docker.com/account/settings/)
Setp2:  选择[Linked Accounts & Services](https://hub.docker.com/account/authorized-services/) ,然后点击link github或者link bitbucket
Setp3:  github账号link完成后，点击用户名旁边的create菜单，选择Create Automated Build,在弹出的页面中选择Create Auto-build
Setp4:  此时会列出Setp2中link的账号中的所有仓库,你从中选择你要Auto Build的仓库(这个仓库中应该放着build镜像所需要的dockerfile和一些文件)
Setp5:  选择完成后跳转到Automated build的设置页面，点击Build Settings，设置Build,设置Build的branch，设置Build采用的dockerfile的位置
Setp6:  此后对github上的dockerfile有改动都会触发docker hub进行Automated Build


## Tips4 docker镜像层的个数有限制吗?这个限制和所选择的Storage Driver有关系吗?
一个镜像是不能超过127层，无论是使用何种Storgae Driver.因此在写Dockfile的时候，应该尽可能的优化Dockrfile中指令的使用，避免产生不必要的镜像层


## Tips5 什么是Image Digests?
docker images在使用v2版本的格式存储的时候，会给镜像添加一个可以基于内容寻址的标识符叫做digest，可以通过`sudo docker images --digests`来查看
如果你的镜像是从v1格式的镜像仓库中拉取的那么对应的digest则是`<none>`，有了digest后，我们就可以使用digest作为标识来运行容器，删除镜像等操作

## Tips6 什么是docker的link机制，引入自定义网络后link机制有什么变化?
参见[<<深入理解docker的link机制>>](http://blog.csdn.net/zhangyifei216/article/details/50921215)

## Tips7 数据卷类型和生命周期
数据卷的类型有如下几种:

* 宿主机上的一个随机目录作为卷挂载到容器中
* 宿主机上指定一个目录作为卷挂载到容器中
* 将一个容器挂载一个宿主机上的随机目录，这个容器可以被其它容器进行挂载

数据卷的生命

## Tips8 通过systemd机制让容器可以自启动

```
[Unit]
Description=Redis container
Requires=docker.service
After=docker.service
 
[Service]
Restart=always
ExecStart=/usr/bin/docker start -a redis_server
ExecStop=/usr/bin/docker stop -t 2 redis_server
  
[Install]
WantedBy=local.target
```
redis_server是你要启动的容器名字，-a标志可以将容器的输入输出重定向到systemd所管理的日志中，这样可以方便使用systemd相关命令查看
-t 2则是在停止一个容器之前等待的时间，可以优雅的关闭容器。将上面的这个文件命名为`<container_name>.service`放到`/usr/lib/systemd/system/`下即可
然后通过systemctl命令来管理这个服务。

## Tips9 如何设置容器内的dns服务器地址
`--dns --dns-search, or --dns-opt`　在运行容器的时候指定这三个操作可以自定义dns服务器地址，dns搜索的domain，dns服务器地址搜索的options
如果没有指定上述选项那么启动容器的时候会搜索宿主机机器上的`/etc/reslov.conf`，如果发现条目就会将其载入到容器中的`/etc/reslov.conf`，否则就使用`8.8.8.8`
和`8.8.4.4`，在容器运行过程中，如果host机器的`/etc/reslov.conf`发生了改变，那么当这个容器stop后再次start后就会自动更新`/etc/reslov.conf`其实现机制是通过
linux kernel's的intofy特性实现的．但是正在运行中的容器是无法自动更新的，需要stop后再次start才行．但是目前这个特性不支持overlay文件系统，因此如果你
的镜像存储用的是overlay文件系统，那么你就无法自动更新dns服务器地址了，如果你使用的是自定义网络，那么默认使用的是docker daemon内置的dns服务，所以`/etc/reslov.conf`
会指向`127.0.0.11`．其实现机制是通过在容器的网络命名空间中设置ipatbles规则将`127.0.0.11 53`端口的tcp/upd访问做了DNAT转到tcp的`127.0.0.1:53710`和udp的`127.0.0.1:37897`上,
然后在这个容器的网络命名空间中启动了内置dns服务。并监听在tcp的53710端口和udp的37897端口。

## Tips10 如何查看容器的网络命名空间
每一个网络命名空间都有一个唯一的标识，在/proc/$PID/ns/net中为了可以通过ip netns命令来访问网络命名空间，必须要将`/proc/$PID/ns/net`链接到`/var/run/netns`目录下
才可以通过ip netns show来查看到命名空间。

## Tips11 如何选择storage driver
参见[docker storage driver compare](http://blog.csdn.net/zhangyifei216/article/details/50697855)

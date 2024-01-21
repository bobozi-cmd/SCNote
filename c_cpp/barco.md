# barco
`barco` 是一个c语言的简单容器实现，基于Linux的内核功能完成其功能。链接：[barco](https://github.com/lucavallin/barco)

### <span style="border-bottom:2px dashed yellow;">The First Glance</span>

从  `barco.c` 的代码中可以清楚的看到，启动一个容器所需的步骤：
- 检查是否以root权限运行：`getuid() != 0`
- 设置hostname，后面还用于标识cgroups目录：`config.hostname`
- 初始化套接字对用于容器内外通信：`socketpair`，这里需要设置套接字fd为`close-on-exec`，以防容器中执行的程序获得此fd：`fcntl(sockets[0], F_SETFD, FD_CLOEXEC)`
- 为容器分配运行栈：`stack = malloc(CONTAINER_STACK_SIZE)`
- 初始化容器、cgroups、namespace，之后会进一步分析：`container_init`、`cgroups_init`、`user_namespace_prepare_mappings`
- 等待容器退出，进行资源回收：`container_wait`

运行一个简单的bash的容器：`sudo ./bin/barco -u 0 -m / -c /bin/bash -a .`，在没退出前，可以看到cgroups目录下有一个`/sys/fs/cgroup/barcontainer`目录，用来给容器限制资源，此外，在`/tmp/barco.XXXXXX`目录下会挂载一个临时目录，这个就是容器内部的fs的根目录。

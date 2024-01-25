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

### <span style="border-bottom:2px dashed yellow;">Container Functions</span>

`int container_init(container_config *config, char *stack)`：创建容器子进程
- clone()启动容器子进程：`container_pid =
           clone(container_start, stack, flags | SIGCHLD, config)`
- 关闭父进程的socket pair[1]：`config.fd = sockets[1] -> close(config->fd) -> config->fd = 0`

`int container_start(void *arg)`：运行容器程序
- 设置hostname：`sethostname(config->hostname, strlen(config->hostname))`
- 挂载目标目录：`mount_set(config->mnt)`
- 初始化用户态命名空间：`user_namespace_init(config->uid, config->fd)`
- 设置capabilities和syscalls：`sec_set_caps() `、` sec_set_seccomp()`
- 关闭子进程的socket pair[1]：`config.fd = sockets[1] -> close(config->fd)`
- 运行程序：`execve(config->cmd, argv, NULL)`

`int container_wait(int container_pid)`：等待容器子进程运行结束，同`waitpid()`

`void container_stop(int container_pid)`：直接杀死容器子进程，同`kill()`

### <span style="border-bottom:2px dashed yellow;">Mount Functions</span>

`int mount_set(char *mnt)`：设置挂载点
- 将文件系统重新挂载为私有，使其在原始命名空间外不可见：`mount(NULL, "/", NULL, MS_REC | MS_PRIVATE, NULL)`
- 在临时文件夹中创建一个临时目录，将指定目录绑定到该临时目录：`mkdtemp(mount_dir) -> mount(mnt, mount_dir, NULL, MS_BIND | MS_PRIVATE, NULL)`
- 创建一个内部临时目录，并使用`pivot_root`系统调用将临时目录作为新的根目录进行切换：`mkdtemp(inner_mount_dir) -> pivot_root(mount_dir, inner_mount_dir)`
- 切换至根目录并对旧的根目录进行卸载操作：`chdir("/") -> umount2(old_root, MNT_DETACH)`
- 清除临时目录和释放资源：`rmdir(old_root)`
- P.S. `pivot_root()`函数允许调用者切换到一个新的根文件系统，并将旧的根挂载点`put_old`放置在`new_root`下的一个位置，从而可以随后卸载它。（它将所有具有旧根目录或当前工作目录的进程移动到新根，使得旧根挂载点可以更容易地卸载，从而释放用户。）

### <span style="border-bottom:2px dashed yellow;">Namespace Functions</span>

`int user_namespace_init(uid_t uid, int fd)`：此函数被容器子进程调用，用于初始化用户命名空间

- 创建一个新的用户命名空间：`int unshared = unshare(CLONE_NEWUSER)`，`unshare()`系统调用用于将当前进程和所在的`Namespace`分离，并加入到一个新的`Namespace`中，该调用会自动创建一个新的`Namespace`
- 通过套接字与父进程通信，告知用户命名空间已启动
- 更新用户和组ID的映射关系：`setgroups(1, &(gid_t){uid}) || setresgid(uid, uid, uid) ||
      setresuid(uid, uid, uid)`

`int user_namespace_prepare_mappings(pid_t pid, int fd)`：此函数用于父进程调用，监听子进程的请求，更新用户和组id的映射关系
- 从套接字fd中读取unshared的值，这是子进程请求设置uid/gid的信号：`read(fd, &unshared, sizeof(unshared)) != sizeof(unshared)`
- 如果unshared的值为0，表示用户命名空间已启用，那么函数将根据子进程的pid创建"/proc/pid/uid_map"和"/proc/pid/gid_map"文件，并更新相应的映射关系。
- 函数通过套接字向子进程发送0来通知子进程映射更新完成。

### <span style="border-bottom:2px dashed yellow;">Seccomp Functions</span>

`int sec_set_caps(void)`：用于设置进程的capabilities权限
- 定义了一个包含需要丢弃的capabilities的数组drop_caps
- 使用prctl函数，将bounding set和inheritable set中需要丢弃的capabilities移除：`prctl(PR_CAPBSET_DROP, drop_caps[i], 0, 0, 0)` 和 `(caps = cap_get_proc()) ||
      cap_set_flag(caps, CAP_INHERITABLE, num_caps, drop_caps, CAP_CLEAR) ||
      cap_set_proc(caps)`
- 释放capabilities的数组：`cap_free(caps)`

`int sec_set_seccomp(void)`：设置 Docker 默认的 seccomp 策略，并且阻止其他过时或危险的系统调用



### <span style="border-bottom:2px dashed yellow;">Cgroups Functions</span>

`int cgroups_init(char *hostname, pid_t pid)`：将指定的进程加入到一个cgroup中，并为该cgroup设置一些资源限制
---
title: '容器镜像'
date: 2021-06-10T17:18:05+01:00
draft: false
---



> 因为招人不顺利的原因，开始在公司内做容器及K8S开发的系统性培训，从容器基础开始也是对自己知识结构的总结，有兴趣加入我们的小伙伴，请私信或留言，坐标重庆。

# 镜像部分

### 概念
- **bootfs**
- **rootfs**
- **layer**
- **union file system**
#### bootfs(boot file system)

主要包含bootloader和kernel, bootloader主要是引导加载kernel, Linux刚启动时会加载bootfs文件系统，在Docker镜像的最底层是bootfs。这一层与我们典型的Linux/Unix系统是一样的，包含boot加载器和内核。当boot加载完成之后整个内核就都在内存中了，此时内存的使用权已由bootfs转交给内核，此时系统也会卸载bootfs。

#### rootfs
rootfs 是挂载在容器根目录上、用来为容器进程提供隔离后执行环境的文件系统，这个文件系统是一个操作系统的**完整文件系统**。需要明确的是，**rootfs 只是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。**

长这样。
```shell
$ ls /
bin dev etc home lib lib64 mnt opt proc root run sbin sys tmp usr var
```
容器的本质
1. 启用 Linux Namespace 配置；
2. 设置指定的 Cgroups 参数；
3. 切换进程的根目录（优先使用 pivot_root 系统调用, 如果系统不支持，使用Change Root）。


> 思考？：rootfs 可以看做一个最基础的操作系统。此时如果要运行java程序，就需要在这个基础的操作系统里安装jre。rootfs 就被污染了，成为了jre 特定的基础操作系统，如果另一个人需要运行python 程序怎么办？重新制作rootfs？

#### Layer
如果修改都基于一个基础的 rootfs，而将变更做成增量，所有人都只需要维护相对于 base rootfs 修改的增量内容，而不是每次修改都制造一个“fork”。

docker 的创新在于用户制作镜像的每一步操作，都会生成一个层，也就是一个增量。
![image.png](/images/layer1.jpg)

我们可以通过 `docker image inspect` 来查看镜像的层
```shell
root@ubuntudev ~# docker image inspect golang:1.16

...
  "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:688e187d6c79c46e8261890f0010fd5d178b8faa178959b0b46b2635aa1eeff3",
                "sha256:00bcea93703b384ab61d2344d6e721c75b0e47198b7912136a86759d8d885711",
                "sha256:ccb9b68523fdb47816f2a15d7590d06ff3d897dfdb3e0733e9c06a2bb79ebfc7",
                "sha256:685934357c8993799eda4a6d71c2bb2d2c54b1138d6e037f675eeeeffc719f2d",
                "sha256:9d52e952d0a781ee401640f2a7a9155f313c927ba5b3880959e055619a3391a9",
                "sha256:762eb5b089c5c496c946d1ffb5d8088a4b3b4648635fd46e8e4c82ee36c33687",
                "sha256:c92e53084342a99bee2b3d5a410f5f6d4df192eff4a8381c5c558cb0f150646d"
            ]
        },
...

```
每一层的变更，都存放在`/var/lib/docker/overlay2/<sha256>`


##### 层的可读写性
![image.png](/images/rw.jpg)

容器（container）的定义和镜像（image）几乎一模一样，也是一堆层的统一视角，唯一区别在于容器的最上面那一层是可读可写的。

> 思考：bin etc 这些操作系统目录被拉平到根目录下，如何还能正常工作？

#### OverLay File System

Overlayfs是一种堆叠文件系统，它依赖并建立在其它的文件系统之上（例如ext4fs和xfs等等），并不直接参与磁盘空间结构的划分，仅仅将原来底层文件系统中不同的目录进行“合并”，然后向用户呈现。因此对于用户来说，它所见到的overlay文件系统根目录下的内容就来自挂载时所指定的不同目录的“合集”。


感受一下
```shell

mkdir -p lower1
mkdir -p lower2
mkdir -p lower3
mkdir -p merge
mkdir -p upper
mkdir -p work

echo "lower1" > lower1/share.txt
echo "lower2" > lower2/share.txt
echo "lower3" > lower3/share.txt
echo "worksh1" > lower1/test1.sh
echo "worksh2" > lower2/test2.sh
echo "worksh3" > lower3/test3.sh
echo "uppersh" > upper/test2.sh
echo "uppertxt" > upper/up.txt

# mount -t overlay overlay -o lowerdir=lower1:lower2:lower3,upperdir=upper,workdir=work merged
# mount -l
```

解释下mount命令各参数含义：
- -t overlay 指定类型为overlay
- -o 后面是挂载选项，指定我们要挂载哪些目录
- lowerdir=xxx：指定用户需要挂载的lower层目录（支持多lower，最大支持500层）；
- upperdir=xxx：指定用户需要挂载的upper层目录；
- workdir=xxx：指定文件系统的工作基础目录，挂载后内容会被清空，且在使用过程中其内容用户不可见；
- default_permissions：功能未使用；
- redirect_dir=on/off：开启或关闭redirect directory特性，开启后可支持merged目录和纯lower层目录的rename/renameat系统调用；
- index=on/off：开启或关闭index特性，开启后可避免hardlink copyup broken问题。

其中lowerdir、upperdir和workdir为基本的挂载选项，redirect_dir和index涉及overlayfs为功能支持选项，除非内核编译时默认启动，否则默认情况下这两个选项不启用，这里先按照默认情况进行演示分析，后面这两个选项单独说明。



[参考](https://blog.csdn.net/luckyapple1028/article/details/77916194)



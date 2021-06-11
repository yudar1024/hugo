---
title: '镜像'
date: 2019-07-18T17:18:05+01:00
draft: false
baseURL: "https://hugo-g723bghqe-yudar1024.vercel.app/"
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
![image.png](https://upload-images.jianshu.io/upload_images/5120230-f52e8b7109a32edd.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

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
![image.png](https://upload-images.jianshu.io/upload_images/5120230-4b8774ec4d9b4c8c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

容器（container）的定义和镜像（image）几乎一模一样，也是一堆层的统一视角，唯一区别在于容器的最上面那一层是可读可写的。

> 思考：bin etc 这些操作系统目录被拉平到根目录下，如何还能正常工作？

#### Union File System
就是用来解决这样一个问题。
UnionFS (联合文件系统) 是一种分层、轻量级并且高性能的文件系统，它支持对文件系统的修改作为一’ 次提交来一层层的叠加，同时可以将不同月录挂载到同一个虚拟文件系统下(unite several directories into a single virtual filesystem)。Union 文件系统是Docker镜像的基础。镜像可以通过分层来进行继承，基于基础镜像(没有父镜像)，可以制作各种具体的应用镜像。

特性:一次同时加载多个文件系统，但从外面看起来，只能看到一个文件系统，联合加载会把各层文件系统叠加起来，这样最终的文件系统会包含所有底层的文件和目录

感受一下
```shell
cd /tmp
mkdir a b c
echo "aaa" > a/a.txt
echo "bbb" > b/b.txt
mount -v -t aufs -o br=/tmp/a:/tmp/b none /tmp/c
ls /tmp/c
```

解释下mount命令各参数含义：
-  -t aufs 指定文件系统类型为aufs
- -o 后面是挂载选项，指定我们要挂载哪些目录
- none 说明我们挂载的不是设备文件，因为这里我们是直接挂载目录的

AUFS的检测级别可以通过udba指定

**udba有三种级别:none、reval、inotify，对性能的影响依次增加，当然安全性也有所增强。**

- None: 这种检测是最快的，但可能导致错误的数据，例如在原始目录修改文件，在aufs中读取，不完全保证正确

- reval：aufs会访问重新原始目录，如果文件有更新，在会反映在aufs中

- Notify： 会在所有原始目录中的所有目录上注册notify事件，这会严重的影响性能，不建议使用。
```shell
mount -v -t aufs -o br=/tmp/a:/tmp/b -o udba=none none /tmp/c
```

**设置读写**
```
mount -v -t aufs -o br=/tmp/a=rw:/tmp/b=ro -o udba=none none /tmp/c
```
[参考](https://blog.csdn.net/lihm0_1/article/details/42030169)



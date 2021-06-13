+++
author = "陈sir"
title = "镜像的存储结构"
date = "2021-06-13"
description = "镜像的存储结构"
tags = [
    "docker",
    "容器",
]
categories = [
    "docker",
    "容器",
]
# image = "images/article.jpg"
+++

[toc]

# 镜像的存储结构

我们先来看看当我们拉取镜像时，流程是什么样的

![进行拉取流程](imgpullflow.jpg)

## 核心概念
- [Image Manifest](https://github.com/opencontainers/image-spec/blob/master/manifest.md)
- [Image Configuration](https://github.com/opencontainers/image-spec/blob/master/config.md)


### OCI Image Manifest

镜像清单(manifest)为特定体系结构和操作系统的单个容器映像提供了配置和一组层。manifes是针对registry服务端的配置信息

例子
``` json
{
  "schemaVersion": 2,
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "size": 7023,
    "digest": "sha256:b5b2b2c507a0944348e0303114d8d93aaaa081732b86451d9bce1f432a537bc7"
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 32654,
      "digest": "sha256:9834876dcfb05cb167a5c24953eba58c4ac89b1adf57f28f2f9d09af107ee8f0"
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 16724,
      "digest": "sha256:3c3a4604a545cdc127456d94e421cd355bca5b528f4a9c1905b15da2eb4a4c6b"
    },
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "size": 73109,
      "digest": "sha256:ec4b8955958665577945c89419d1af06b5f7636b4ac3da7f12184802ad867736"
    }
  ],
  "annotations": {
    "com.example.key1": "value1",
    "com.example.key2": "value2"
  }
}
```

#### digest

digest是对manifest文件的sha256摘要，当镜像的内容变化时，即layer变化，相应的layer的sha256变化，以至manifest变化，从而保证了一个digest（不是镜像名+tag）对应一个镜像。 `一个digest 对应一个image id`

> 通过 digest 解决了相同镜像名+tag 得到不同镜像的问题。 

### OCI Image Configuration

OCI Image是根文件系统变更和容器运行时中使用的执行参数的有序集合。该规范概述了用于描述容器运行时和执行程序的镜像的JSON格式，以及它与文件系统变更集的关系，如Layers所述。

样例
```json
{
    "created": "2015-10-31T22:22:56.015925234Z",
    "author": "Alyssa P. Hacker <alyspdev@example.com>",
    "architecture": "amd64",
    "os": "linux",
    "config": {
        "User": "alice",
        "ExposedPorts": {
            "8080/tcp": {}
        },
        "Env": [
            "PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin",
            "FOO=oci_is_a",
            "BAR=well_written_spec"
        ],
        "Entrypoint": [
            "/bin/my-app-binary"
        ],
        "Cmd": [
            "--foreground",
            "--config",
            "/etc/my-app.d/default.cfg"
        ],
        "Volumes": {
            "/var/job-result-data": {},
            "/var/log/my-app-logs": {}
        },
        "WorkingDir": "/home/alice",
        "Labels": {
            "com.example.project.git.url": "https://example.com/project.git",
            "com.example.project.git.commit": "45a939b2999782a3f005621a8d0f29aa387e1d6b"
        }
    },
    "rootfs": {
      "diff_ids": [
        "sha256:c6f988f4874bb0add23a778f753c65efe992244e148a1d2ec2a8b664fb66bbd1",
        "sha256:5f70bf18a086007016e948b04aed3b82103a36bea41755b6cddfaf10ace3c6ef"
      ],
      "type": "layers"
    },
    "history": [
      {
        "created": "2015-10-31T22:22:54.690851953Z",
        "created_by": "/bin/sh -c #(nop) ADD file:a3bc1e842b69636f9df5256c49c5374fb4eef1e281fe3f282c65fb853ee171c5 in /"
      },
      {
        "created": "2015-10-31T22:22:55.613815829Z",
        "created_by": "/bin/sh -c #(nop) CMD [\"sh\"]",
        "empty_layer": true
      }
    ]
}
```
##  docker镜像在本地磁盘上的存储
先看看 docker 的根目录以及存储配置

```shell
root@ubuntudev ~# docker info

...

 Server Version: 20.10.7
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 1
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: runc io.containerd.runc.v2 io.containerd.runtime.v1.linux
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: d71fcd7d8303cbf684402823e425e9dd2e99285d
 runc version: b9ee9c6314599f1b4a7f497e1f1f856fe433d3b7
 init version: de40ad0
 Security Options:
  apparmor
  seccomp
   Profile: default
 Kernel Version: 5.4.0-74-generic
 Operating System: Ubuntu 20.04.2 LTS
 OSType: linux
 Architecture: x86_64

 ...
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Registry: https://index.docker.io/v1/
 Labels:
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
 ```

 `Docker Root Dir: /var/lib/docker` 和 ` Storage Driver: overlay2` 标明了docker 在本地的存储位置以及使用的存储驱动类型

 而镜像内部的存储结构，是下面这个鬼样子
 
 ![镜像存储结构](imagelocalstore.jpg)

### docker目录结构
```shell
root@ubuntudev ~# ll /var/lib/docker
total 44K
drwx--x--x  4 root root 4.0K Jun  7 15:49 buildkit
drwx-----x  4 root root 4.0K Jun 10 16:29 containers
drwx------  3 root root 4.0K Jun  7 15:49 image
drwxr-x---  3 root root 4.0K Jun  7 15:49 network
drwx-----x 49 root root 4.0K Jun 10 16:29 overlay2
drwx------  4 root root 4.0K Jun  7 15:49 plugins
drwx------  2 root root 4.0K Jun  8 13:44 runtimes
drwx------  2 root root 4.0K Jun  7 15:49 swarm
drwx------  2 root root 4.0K Jun 10 16:29 tmp
drwx------  2 root root 4.0K Jun  7 15:49 trust
drwx-----x  3 root root 4.0K Jun  8 13:44 volumes
```

- overlay2: 镜像和容器的层信息
- image：存储镜像元相关信息

### image 目录结构

#### repositories.json
repositories.json就是存储镜像信息，主要是name和image id的对应，digest和image id的对应。当pull镜像的时候会更新这个文件。


```shell
root@ubuntudev ~ [127]# cat /var/lib/docker/image/overlay2/repositories.json | python3 -m json.tool
{
    "Repositories": {
       
        "golang": {
            "golang:1.16": "sha256:b09f7387a7195b1cfe0144557a8e33af2174426a4b76cb89e499093803d02e7b",
            "golang@sha256:360bc82ac2b24e9ab6e5867eebac780920b92175bb2e9e1952dce15571699baa": "sha256:b09f7387a7195b1cfe0144557a8e33af2174426a4b76cb89e499093803d02e7b"
        },
        "nginx": {
            "nginx:latest": "sha256:d1a364dc548d5357f0da3268c888e1971bbdb957ee3f028fe7194f1d61c6fdee",
            "nginx@sha256:6d75c99af15565a301e48297fa2d121e15d80ad526f8369c526324f0f7ccb750": "sha256:d1a364dc548d5357f0da3268c888e1971bbdb957ee3f028fe7194f1d61c6fdee"
        } 
    }
}
```

#### imagedb

image id和digest的联系：

```shell
root@ubuntudev ~ [125]# docker images --digests

...

golang                                                        1.16      sha256:360bc82ac2b24e9ab6e5867eebac780920b92175bb2e9e1952dce15571699baa   b09f7387a719   9 days ago      862MB
nginx                                                         latest    sha256:6d75c99af15565a301e48297fa2d121e15d80ad526f8369c526324f0f7ccb750   d1a364dc548d   2 weeks ago     133MB
```

```shell
root@ubuntudev ~# docker manifest inspect -v nginx
[
        {
                "Ref": "docker.io/library/nginx:latest@sha256:61191087790c31e43eb37caa10de1135b002f10c09fdda7fa8a5989db74033aa",
                "Descriptor": {
                        "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
                        "digest": "sha256:61191087790c31e43eb37caa10de1135b002f10c09fdda7fa8a5989db74033aa",
                        "size": 1570,
                        "platform": {
                                "architecture": "amd64",
                                "os": "linux"
                        }
                },
                "SchemaV2Manifest": {
                        "schemaVersion": 2,
                        "mediaType": "application/vnd.docker.distribution.manifest.v2+json",
                        "config": {
                                "mediaType": "application/vnd.docker.container.image.v1+json",
                                "size": 7732,
                                "digest": "sha256:d1a364dc548d5357f0da3268c888e1971bbdb957ee3f028fe7194f1d61c6fdee"
                        },
                        "layers": [
                                {
                                        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
                                        "size": 27145915,
                                        "digest": "sha256:69692152171afee1fd341febc390747cfca2ff302f2881d8b394e786af605696"
                                },
                                {
                                        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
                                        "size": 26580149,
                                        "digest": "sha256:30afc0b18f67ae8441c2d26e356693009bb8927ab7e3bce05d5ed99531c9c1d4"
                                },
                                {
                                        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
                                        "size": 601,
                                        "digest": "sha256:596b1d696923618bec6ff5376cc9aed03a3724bc75b6c03221fd877b62046d05"
                                },
                                {
                                        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
                                        "size": 893,
                                        "digest": "sha256:febe5bd23e98102ed5ff64b8f5987f516a945745c08bbcf2c61a50fb6e7b2257"
                                },
                                {
                                        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
                                        "size": 665,
                                        "digest": "sha256:8283eee92e2f756bd57f96ea295e332ab9031724267d4f939de1f7d19fe9611a"
                                },
                                {
                                        "mediaType": "application/vnd.docker.image.rootfs.diff.tar.gzip",
                                        "size": 1394,
                                        "digest": "sha256:351ad75a6cfabc7f2e103963945ff803d818f0bdcf604fd2072a0eefd6674bde"
                                }
                        ]
                }
        },
        ....
]
```

#### distribution

我们在看看镜像中的layer存的是什么
```shell
root@ubuntudev ~# docker image inspect nginx
[
    {
        "Id": "sha256:d1a364dc548d5357f0da3268c888e1971bbdb957ee3f028fe7194f1d61c6fdee",
        "RepoTags": [
            "nginx:latest"
        ],
        "RepoDigests": [
            "nginx@sha256:6d75c99af15565a301e48297fa2d121e15d80ad526f8369c526324f0f7ccb750"
        ],
...
        "GraphDriver": {
            "Data": {
                "LowerDir": "/var/lib/docker/overlay2/23063c7072b8ecb7484c9a5981adc07618ff91d28e379343a045a6ae99dfd53d/diff:/var/lib/docker/overlay2/aa1ac4e25288bca64b6d318684fb5b762f8687268ef9471ac4fb932e66963ba5/diff:/var/lib/docker/overlay2/a689f489f0528bfdfccb022ff4745e3d8ff5749beaeebca3f3cdc8222262b452/diff:/var/lib/docker/overlay2/dea3cfea8d34b61b9073c57e22e18671aa80ab97ebec6fc3d6e56d33c19b77c3/diff:/var/lib/docker/overlay2/8cc45c9b7a0e614745ca1f6b264b39a637d2c77910da368ae7cd10fa9a0bfff0/diff",
                "MergedDir": "/var/lib/docker/overlay2/2ab5d68cf289b00200e349d8ab0e5b94941a7c5c396a4ed17768f866450c93e8/merged",
                "UpperDir": "/var/lib/docker/overlay2/2ab5d68cf289b00200e349d8ab0e5b94941a7c5c396a4ed17768f866450c93e8/diff",
                "WorkDir": "/var/lib/docker/overlay2/2ab5d68cf289b00200e349d8ab0e5b94941a7c5c396a4ed17768f866450c93e8/work"
            },
            "Name": "overlay2"
        },
        "RootFS": {
            "Type": "layers",
            "Layers": [
                "sha256:02c055ef67f5904019f43a41ea5f099996d8e7633749b6e606c400526b2c4b33",
                "sha256:766fe2c3fc083fdb0e132c138118bc931e3cd1bf4a8bdf0e049afbf64bae5ee6",
                "sha256:83634f76e73296b28a0e90c640494970bdfc437749598e0e91e77eea9bdb6a4e",
                "sha256:134e19b2fac580eff84faabfd5067977b79e36c5981d51fd63e8ac752dbdf9ec",
                "sha256:5c865c78bc96874203b5aa48f1a089d1eabcbe1607edaa16aaa6dee27c985395",
                "sha256:075508cf8f04705d8dc648cfb9f044f5dff57c31ccf34bde32cd2874f402dfad"
            ]
        },
        "Metadata": {
            "LastTagTime": "0001-01-01T00:00:00Z"
        }
    }
]
```
发现manifest 和 image config 的 sha256 不一样，这个过程是因为 image config 的sha256 是manifest 中的 压缩包的值。

`curl -H "Accept:application/vnd.docker.image.rootfs.diff.tar.gzip" https://<your-registry>/v2/ubuntu/blobs/sha256:<manifest-sha256>`

这个压缩包 sha256 之后才会得到 image config 中的 sha256 layer 值

![参考](https://zhuanlan.zhihu.com/p/95900321)

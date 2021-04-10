### docker源码打开

1,  安装编辑器 webstrom

2，安装golang

3，配置gopath

```
D:\golang
```

4, 从码云拉取docker 代码

```
cd D:\golang\src
git clone https://gitee.com/mirrors/moby.git
```

5，安装docker依赖

```
直接go mod tidy 下载依赖会报版本异常

删除 vendor 目录 和 vendor.conf 文件，执行 go mod init

在 go mod中 添加

replace (
	github.com/Sirupsen/logrus v1.4.0 => github.com/sirupsen/logrus v1.4.0
	github.com/sirupsen/logrus v1.8.1 => github.com/Sirupsen/logrus v1.8.1
)

```

6,继续安装

```
module github.com/docker/docker

go 1.15

replace (
	github.com/Sirupsen/logrus v1.4.0 => github.com/sirupsen/logrus v1.4.0
	github.com/sirupsen/logrus v1.8.1 => github.com/Sirupsen/logrus v1.8.1
)

require (
	cloud.google.com/go v0.81.0
	cloud.google.com/go/logging v1.3.0
	github.com/BurntSushi/toml v0.3.1
	github.com/Graylog2/go-gelf v0.0.0-20170811154226-7ebf4f536d8f
	github.com/Microsoft/go-winio v0.4.17-0.20210211115548-6eac466e5fa3
	github.com/Microsoft/hcsshim v0.8.16
	github.com/Microsoft/opengcs v0.4.0
	github.com/RackSec/srslog v0.0.0-20180709174129-a4725f04ec91
	github.com/Sirupsen/logrus v1.4.0 // indirect
	github.com/aws/aws-sdk-go v1.38.17
	github.com/boltdb/bolt v1.3.1 // indirect
	github.com/bsphere/le_go v0.0.0-20200109081728-fc06dab2caa8
	github.com/cloudflare/cfssl v1.5.0
	github.com/containerd/cgroups v0.0.0-20210114181951-8a68de567b68
	github.com/containerd/containerd v1.5.0-beta.4
	github.com/containerd/continuity v0.0.0-20210315143101-93e15499afd5
	github.com/containerd/fifo v0.0.0-20210331061852-650e8a8a179d
	github.com/containerd/typeurl v1.0.1
	github.com/coreos/go-systemd/v22 v22.3.1
	github.com/creack/pty v1.1.11
	github.com/deckarep/golang-set v1.7.1 // indirect
	github.com/docker/distribution v2.7.1+incompatible
	github.com/docker/engine-api v0.4.0 // indirect
	github.com/docker/go-connections v0.4.0
	github.com/docker/go-metrics v0.0.1
	github.com/docker/go-units v0.4.0
	github.com/docker/libkv v0.2.1
	github.com/docker/libnetwork v0.8.0-dev.2.0.20200917202933-d0951081b35f
	github.com/docker/libtrust v0.0.0-20160708172513-aabc10ec26b7
	github.com/docker/swarmkit v1.12.0
	github.com/fluent/fluent-logger-golang v1.5.0
	github.com/fsnotify/fsnotify v1.4.9
	github.com/gogo/protobuf v1.3.2
	github.com/golang/gddo v0.0.0-20210115222349-20d68f94ee1f
	github.com/google/go-cmp v0.5.5
	github.com/google/uuid v1.2.0
	github.com/gorilla/mux v1.8.0
	github.com/hashicorp/go-immutable-radix v1.3.0
	github.com/hashicorp/go-memdb v1.3.2
	github.com/imdario/mergo v0.3.12
	github.com/mistifyio/go-zfs v2.1.2-0.20190413222219-f784269be439+incompatible
	github.com/moby/buildkit v0.8.2
	github.com/moby/ipvs v1.0.1 // indirect
	github.com/moby/locker v1.0.1
	github.com/moby/sys/mount v0.2.0
	github.com/moby/sys/mountinfo v0.4.1
	github.com/moby/sys/symlink v0.1.0
	github.com/moby/term v0.0.0-20201216013528-df9cb8a40635
	github.com/morikuni/aec v1.0.0
	github.com/opencontainers/go-digest v1.0.0
	github.com/opencontainers/image-spec v1.0.1
	github.com/opencontainers/runc v1.0.0-rc93
	github.com/opencontainers/runtime-spec v1.0.3-0.20200929063507-e6143ca7d51d
	github.com/opencontainers/selinux v1.8.0
	github.com/pivotal-golang/clock v1.0.0 // indirect
	github.com/pkg/errors v0.9.1
	github.com/prometheus/client_golang v1.10.0
	github.com/sirupsen/logrus v1.8.1
	github.com/spf13/cobra v1.1.3
	github.com/spf13/pflag v1.0.5
	github.com/syndtr/gocapability v0.0.0-20200815063812-42c35b437635
	github.com/tchap/go-patricia v2.3.0+incompatible
	github.com/tinylib/msgp v1.1.5 // indirect
	github.com/tonistiigi/fsutil v0.0.0-20210315162859-fde37e74a709
	github.com/vbatts/tar-split v0.11.1
	github.com/vishvananda/netlink v1.1.0
	go.etcd.io/bbolt v1.3.5
	golang.org/x/net v0.0.0-20210405180319-a5a99cb37ef4
	golang.org/x/sync v0.0.0-20210220032951-036812b2e83c
	golang.org/x/sys v0.0.0-20210403161142-5e06dd20ab57
	golang.org/x/time v0.0.0-20210220033141-f8bda1e9f3ba
	google.golang.org/genproto v0.0.0-20210406143921-e86de6bf7a46
	google.golang.org/grpc v1.37.0
	gotest.tools/v3 v3.0.3
)
```

当然，足够熟练可以直接编辑go.mod,例如直接安装报版本错误

```
go mod edit -replace google.golang.org/grpc@v1.37.0=google.golang.org/grpc@v1.26.0
```



注：logrus 版本一开始不知道是多少添加错了也没关系，下载过程会报错，抛出正确的版本号，在再go.mod中修改重新下载即可.

7，结果体验

如图所示，当看到函数都绿了，不在显示红色，可以检索函数了，就可以愉快的阅读docker源码了.  

![image](https://github.com/Mountains-and-rivers/k8s-code/blob/main/images/docker-01.png)

![image](https://github.com/Mountains-and-rivers/k8s-code/blob/main/images/docker-02.png)

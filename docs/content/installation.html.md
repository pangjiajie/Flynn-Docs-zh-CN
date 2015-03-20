---
title: 安装
layout: docs
---

# 安装

开始前，你应通过以下命令安装 Flynn 的命令行：

```bash
L=/usr/local/bin/flynn && curl -sL -A "`uname -sp`" https://cli.flynn.io/flynn.gz | zcat >$L && chmod +x $L
```

如果你想在本机上运行 Flynn， 最简单的方式是安装 [Vagrant demo environment](#vagrant)。

如果你想部署 Flynn 到 AWS，[你可以使用 CLI](#aws)。

如果你想手动安装 Flynn，参照 [Ubuntu 14.04 amd64](#ubuntu-14.04-amd64) 指南。现在只有 Ubuntu 14.04 amd64 被支持，不过这仅是一个暂时的软件包限制，我们并没有实际的对于 Ubuntu 的依赖。

## Vagrant

### 依赖

需要安装 Vagrant 和 VirtualBox，如果没有，请参照其网站上的指南安装：

* [VirtualBox](https://www.virtualbox.org/)
* [Vagrant 1.6 or greater](http://www.vagrantup.com/)

### 安装

Clone the Flynn git repository:

```
$ git clone https://github.com/flynn/flynn
```

Change to the `demo` directory and bring up the Vagrant box:

```
$ cd flynn/demo
$ vagrant up
```

If the VM fails to boot for any reason, you can restart the process by running the following:

```
$ vagrant reload
```

With a successful installation, you will have a single node Flynn cluster running inside the VM,
and the final log line contains a `flynn cluster add` command. Paste that line from the console
output into your terminal and execute it.

Now you have Flynn installed and running, head over to the [Using Flynn](/docs)
page for guides on deploying your applications to Flynn.


## AWS

The [Flynn CLI](https://cli.flynn.io) includes an installer that will boot and
configure a Flynn cluster on Amazon Web Services using CloudFormation. It
automatically performs all of the steps required to install Flynn.

Just run `flynn install aws` to start the installation. You may [optionally
specify](/docs/cli#install) the region, instance type, and number of instances.


## Ubuntu 14.04 amd64

Before we get going with the installation, please note that if you plan on running a multi-node
cluster, you should boot at least 3 nodes to keep etcd efficient
(see [here](https://github.com/coreos/etcd/blob/v0.4.6/Documentation/optimal-cluster-size.md) for
an explanation).

*NOTE: If you are installing on Linode, you need to use native kernels (rather than
Linode kernels) for AUFS support, see [this guide](https://www.linode.com/docs/tools-reference/custom-kernels-distros/run-a-distributionsupplied-kernel-with-pvgrub)
for instructions on how to switch.*

### Installation

Download and run the Flynn installer script:

```
$ sudo bash < <(curl -fsSL https://dl.flynn.io/install-flynn)
```

If you would rather take a look at the contents of the script before running it as root, download and
run it in two separate steps:

```
$ curl -fsSL -o /tmp/install-flynn https://dl.flynn.io/install-flynn
... take a look at the contents of /tmp/install-flynn ...
$ sudo bash /tmp/install-flynn
```

Running the installer script will:

1. Install Flynn's runtime dependencies
2. Download, verify and install the `flynn-host` binary
3. Download and verify filesystem images for each of the Flynn components
4. Install an Upstart job for controlling the `flynn-host` daemon

Some of the filesystem images are quite large (hundreds of MB) so step 3 could take a while depending on
your internet connection.

### Rinse and repeat

You should install Flynn as above on every host that you want to be in the Flynn cluster.
你应该如上安装 Flynn 在每个你想加入的 Flynn 集群中。

### Start Flynn Layer 0

First, ensure that all network traffic is allowed between all nodes in the cluster (specifically
all UDP and TCP packets). The following ports should also be open externally on the firewalls
for all nodes in the cluster:
首先，确保所有的网络流量允许在集群的所有节点中传递（尤其是所有的 UDP 和 TCP 包）。 下列端口也应该在所有集群节点的外部或防火墙上暴露。

* 80 (HTTP)
* 443 (HTTPS)
* 2222 (Git over SSH)
* 3000 to 3500 (user defined TCP services)

The next step is to configure a Layer 0 cluster by starting the flynn-host daemon on all
nodes. The daemon uses etcd for leader election, and etcd needs to be aware of all of the
other nodes for it to function correctly.

下一步是在第0层部署集群，通过在所有节点上启动 flynn-host 进程。该进程使用 etcd 来进行主从选举，并且 etcd 需要知晓所有其他节点来使它正确工作。

If you are starting more than one node, the etcd cluster should be configured
using a [discovery
token](https://coreos.com/docs/cluster-management/setup/etcd-cluster-discovery/).
`flynn-host init` is a tool that handles generating and configuring the token.

如果你使用了超过一个节点，那么 etcd 集群应该使用 [discovery
token](https://coreos.com/docs/cluster-management/setup/etcd-cluster-discovery/) 来配置。

`flynn-host init` 是用来产生和配置 token 的工具。

On the first node, create a new token with the `--init-discovery=3` flag,
replacing `3` with the total number of nodes that will be started. The minimum
multi-node cluster size is three, and this command does not need to be run if
you are only starting a single node.

在第一个节点上，生成一个带有 `--init-discovery=3` 的 token，将 `3` 替换成你将启动的节点数目。最小的多节点系统的数量是 3 个。如果仅需一个节点，那么无需使用此 命令。 

```
$ sudo flynn-host init --init-discovery=3
https://discovery.etcd.io/ac4581ec13a1d4baee9f9c78cf06a8c0
```

On the rest of the nodes, configure the generated discovery token:

在其余节点处，配置生成的 discovery token：

```
$ sudo flynn-host init --discovery https://discovery.etcd.io/ac4581ec13a1d4baee9f9c78cf06a8c0
```

**Note:** a new token must be used every time you restart all nodes in the
cluster.

**Note:** 每次你重启集群中的所有节点时，你必须使用一个新的 token。

Then, start the daemon by running:
接着，开启进程：

```
$ sudo start flynn-host
```

You can check the status of the daemon by running:
你可以检查进程的状态：

```
$ sudo status flynn-host
flynn-host start/running, process 4090
```

If the status is `stop/waiting`, the daemon has failed to start for some reason. Check the
log file (`/var/log/upstart/flynn-host.log`) for any errors and try starting the daemon
again.

如果状态是 `stop/waiting`，那么进程因某些原因运行失败。。检查 log 文件 (`/var/log/upstart/flynn-host.log`) 来找出错误并尝试重启进程。

### Start Flynn Layer 1

After you have a running Layer 0 cluster, you can now bootstrap Layer 1 with
`flynn-host bootstrap`. You'll need a domain name with DNS A records pointing to
every node IP address and a second, wildcard domain CNAME to the cluster domain.

在运行了 Layer 0 集群之后，你可以通过执行 `flynn-host bootstrap` 启动 Layer 1。你需要一个域名和 DNS 的A记录来指向每一个节点的 IP 地址，和一个泛域名 CNMAE 记录指向集群域名。

**Example**

```
demo.localflynn.com.    A      192.168.84.42
demo.localflynn.com.    A      192.168.84.43
demo.localflynn.com.    A      192.168.84.44
*.demo.localflynn.com.  CNAME  demo.localflynn.com.
```

*If you are just using a single node and don't want to initially setup DNS
records, you can use [xip.io](http://xip.io) which provides wildcard DNS for
any IP address.*

Set `CLUSTER_DOMAIN` to the main domain name and start the bootstrap process,
specifying the number of hosts that are expected to be present.

```
$ sudo \
    CLUSTER_DOMAIN=demo.localflynn.com \
    flynn-host bootstrap --min-hosts=3 /etc/flynn/bootstrap-manifest.json
```

*Note: You only need to run this on a single node in the cluster. It will
schedule jobs on nodes across the cluster as required.*

The Layer 1 bootstrapper will get all necessary services running using the Layer
0 API. The final log line will contain configuration that may be used with the
[command-line interface](/docs/cli).

If you try these instructions and run into issues, please open an issue or pull
request.

Now you have Flynn installed and running, head over to the [Using Flynn](/docs)
page for guides on deploying your applications to Flynn.

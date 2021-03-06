---
title: 使用 Flynn
layout: docs
---

# 使用 Flynn

这篇指南假设你已经有了一个运行中的 Flynn 集群，并且已经配置好了 `flynn` 命令行。如果没有，请参照 [Installation
Guide](/docs/installation) 配置环境。

本教程同时假设你使用 `demo.localflynn.com` 默认域名（如果你安装的是示例环境）。如果你正在使用自己的域名，在引导过程中请将 `demo.localflynn.com` 替换成你设置的 `CLUSTER_DOMAIN`。

## 添加 SSH key

在部署 Flynn 之前，你需要先添加 SSH 公钥：

```
$ flynn key add
Key dd:31:2c:07:33:fb:93:32:2b:cc:fa:87:a4:0f:00:34 added.
```

*查看 [这里](/docs/cli#key) 获取关于 `flynn key` 命令的更多信息。*

## 部署

我们将要部署一个 Node.js 示例程序，它将启动一个最简 HTTP 服务器。

克隆以下 Git repo：

```
$ git clone https://github.com/flynn/nodejs-flynn-example.git
```

在克隆的 repo 中，创建一个 Flynn 应用：

```
$ cd nodejs-flynn-example
$ flynn create example
Created example
```

上述命令应添加一个 `flynn` Git remote：

```
$ git remote -v
flynn   ssh://git@demo.localflynn.com:2222/example.git (push)
flynn   ssh://git@demo.localflynn.com:2222/example.git (fetch)
origin  https://github.com/flynn/nodejs-flynn-example.git (fetch)
origin  https://github.com/flynn/nodejs-flynn-example.git (push)
```

它也应添加一个默认地址 `example.demo.localflynn.com` 来指向 `example-web` 服务：

```
$ flynn route
ROUTE                             SERVICE      ID
http:example.demo.localflynn.com  example-web  http/1ba949d1654e711d03b5f1e471426512
```

推送 `flynn` Git remote 来部署应用：

```
$ git push flynn master
...
-----> Building example...
-----> Node.js app detected
...
-----> Creating release...
=====> Application deployed
=====> Added default web=1 formation
To ssh://git@demo.localflynn.com:2222/example.git
 * [new branch]      master -> master
```

现在应用部署完成，你可以通过应用的默认地址来发送 HTTP 请求。

```
$ curl http://example.demo.localflynn.com
Hello from Flynn on port 55006 from container d55c7a2d5ef542c186e0feac5b94a0b0
```

## Scale 扩展

应用在 root 目录下的 `Procfile` 中声明了它们的进程类型。示例应用定义了一个 `web` 进程类型，并执行 `web.js`：

```
$ cat Procfile
web: node web.js
```

process, as can be seen with the `ps` command:
以 `web` 为进程类型的新应用初始只运行了一个 web 进程，可以通过 `ps` 命令来查看：

```
$ flynn ps
ID                                      TYPE
flynn-d55c7a2d5ef542c186e0feac5b94a0b0  web
```

通过 `scale` 命令来运行更多的 web 进程：

```
$ flynn scale web=3
```

`ps` 应该显示三个运行的进程：

```
$ flynn ps
ID                                      TYPE
flynn-7c540dffaa7e434db3849280ed5ba020  web
flynn-3e8572dd4e5f4136a6a2243eadca5e02  web
flynn-d55c7a2d5ef542c186e0feac5b94a0b0  web
```

重复的 HTTP 请求应该展现出请求由进程负载均衡：

```
$ curl http://example.demo.localflynn.com
Hello from Flynn on port 55006 from container d55c7a2d5ef542c186e0feac5b94a0b0

$ curl http://example.demo.localflynn.com
Hello from Flynn on port 55007 from container 3e8572dd4e5f4136a6a2243eadca5e02

$ curl http://example.demo.localflynn.com
Hello from Flynn on port 55008 from container 7c540dffaa7e434db3849280ed5ba020

$ curl http://example.demo.localflynn.com
Hello from Flynn on port 55007 from container 3e8572dd4e5f4136a6a2243eadca5e02
```

## Logs 日志

你可以通过 `log` 命令查看任务的记录 (i.e. stdout / stderr)：

```
$ flynn log flynn-d55c7a2d5ef542c186e0feac5b94a0b0
Listening on 55006
```

*查看 [这里](/docs/cli#log) 获取更多关于 `flynn log` 命令的信息。*

## 发布

通过 Git 提交变更推送至 Flynn 来发布新的版本。

添加下面这行代码到 `web.js` 的第一行：

```js
console.log("I've made a change!")
```

提交到 Git，push 变更至 Flynn：

```
$ git add web.js
$ git commit -m "Add log message"
$ git push flynn master
```

一旦 push 成功，你将看到3个新进程：

```
$ flynn ps
ID                                      TYPE
flynn-cf834b6db8bb4514a34372c8b0020b1e  web
flynn-16f2725f165343fca22a65eebab4e230  web
flynn-d7893da39a8847f395ce47f024479145  web
```

这些进程的记录应该展示添加的记录信息：

```
$ flynn log flynn-cf834b6db8bb4514a34372c8b0020b1e
I've made a change!
Listening on 55007
```

## Routes

初始创建后，应用会获得一个默认路由，它是根域名的子域名 (e.g. `example.demo.localflynn.com`)。如果你想使用其他域名，你需要添加其他路由。

假如你有一个名为 `example.com` 的域名，它指向你的 Flynn 集群 (e.g. it is a `CNAME` for `example.demo.localflynn.com`).

为该域名添加路由：

```
$ flynn route add http example.com
http/5ababd603b22780302dd8d83498e5172
```

你的应用现在应当有两条路由记录：

```
$ flynn route
ROUTE                             SERVICE      ID
http:example.com                  example-web  http/5ababd603b22780302dd8d83498e5172
http:example.demo.localflynn.com  example-web  http/1ba949d1654e711d03b5f1e471426512
```

指向 `example.com` 的 HTTP 请求应当被路由指向到 web 进程：

```
$ curl http://example.com
Hello from Flynn on port 55007 from container cf834b6db8bb4514a34372c8b0020b1e
```

你现在可以修改你的应用来响应不同的 HTTP Host header  (即可以是 `example.demo.localflynn.com` 也可以是 `example.com`).

## 多进程

目前示例应用只有一个进程类型 (即 `web` 进程)，但应用可以拥有多个进程类型，并且独立扩展。

我们添加一个简单的 `clock` 服务来打印每秒的时间。

将如下代码加入 `clock.js`：

```js
setInterval(function() {
  console.log(new Date().toTimeString());
}, 1000);
```

通过将如下代码添加至 `Procfile` 来创建 `clock` 进程类型：

```
clock: node clock.js
```

发布这些改动：

```
$ git add clock.js Procfile
$ git commit -m "Add clock service"
$ git push flynn master
```

扩展 `clock` 服务至一个进程并显示：

```
$ flynn scale clock=1

$ flynn ps
ID                                      TYPE
flynn-aff3ae71d0c149b185ec64ea2885075f  clock
flynn-cf834b6db8bb4514a34372c8b0020b1e  web
flynn-16f2725f165343fca22a65eebab4e230  web
flynn-d7893da39a8847f395ce47f024479145  web

$ flynn log flynn-aff3ae71d0c149b185ec64ea2885075f
18:40:42 GMT+0000 (UTC)
18:40:43 GMT+0000 (UTC)
18:40:44 GMT+0000 (UTC)
18:40:45 GMT+0000 (UTC)
18:40:46 GMT+0000 (UTC)
...
```

## 运行

可以在容器中创建一个交互的一次性的进程：

```
$ flynn run bash
```

*See [here](/docs/cli#run) for more information on the `flynn run` command.*
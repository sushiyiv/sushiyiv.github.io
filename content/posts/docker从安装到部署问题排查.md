+++
title = 'Docker从安装到部署问题排查'
date = '2026-07-13T22:26:51+08:00'
draft = false
description = ""
tags = []
categories = []
+++

## 概述

现象一：apt install docker.io 失败
原因：  系统 DNS 指向 127.0.0.53（systemd-resolved），未正常工作
修复：  Netplan 配置公共 DNS 223.5.5.5

现象二：docker pull 卡住无输出
原因：  Docker 有独立 DNS 配置，daemon.json 不存在，仍使用 127.0.0.53
修复：  创建 daemon.json，配置 dns: ["223.5.5.5", "8.8.8.8"]

现象三：docker pull 报 not found
原因：  国内 Docker 镜像源失效或不稳定
修复：  多次更换可用镜像源

现象四：docker pull 再次卡住
原因：  IPv6 DNS 污染，docker.io 解析到 Facebook 的 IPv6 地址
修复：  禁用 IPv6（/etc/sysctl.conf 添加 disable_ipv6 = 1）

## 正文

# 故障排错日志：Docker 环境搭建

---

## 基本信息

```
故障标题：Ubuntu 虚拟机安装 Docker 及拉取镜像失败
故障时间：2026-07-09
影响范围：虚拟机内无法正常安装软件和拉取 Docker 镜像
故障时长：约 1 小时
处理人：SU
最终状态：已解决
```

---

## 故障现象

### 现象一：apt 安装 Docker 失败

```
现象：
  执行 sudo apt install docker.io 失败
  报错：Temporary failure resolving 'cn.archive.ubuntu.com'

时间：2026-07-09 18:00
```

### 现象二：Docker 拉取镜像失败

```
现象：
  执行 docker pull nginx
  卡在 Using default tag: latest，没有任何输出

时间：2026-07-09 18:30
```

### 现象三：Docker 拉取镜像报错 not found

```
现象：
  执行 docker pull nginx
  报错：docker.io/library/nginx:latest: not found

时间：2026-07-09 18:45
```

### 现象四：Docker 拉取镜像再次卡住

```
现象：
  执行 docker pull nginx
  显示 Pulling fs layer 后卡住不动

时间：2026-07-09 19:00
```

---

## 排查过程

### 第一步：apt 安装失败排查

```
操作：sudo apt install docker.io
报错：Temporary failure resolving 'cn.archive.ubuntu.com'

分析：
  "Temporary failure resolving" 是 DNS 解析失败
  域名无法解析为 IP 地址

排查命令：
  ping 223.5.5.5      → 能 ping 通（网络正常）
  ping baidu.com       → 无法解析（确认是 DNS 问题）
  cat /etc/resolv.conf → 显示 nameserver 127.0.0.53
                         （systemd-resolved 本地代理，未正常工作）

根因：
  系统 DNS 配置指向 127.0.0.53（systemd-resolved 本地解析器）
  该解析器未正常工作，导致所有域名解析失败
```

**处理：**

```yaml
# 修改 Netplan 配置，添加公共 DNS
sudo vim /etc/netplan/00-installer-config.yaml

# 添加内容
nameservers:
    addresses:
        - 223.5.5.5
        - 8.8.8.8

# 生效
sudo netplan apply
```

```
结果：
  ping baidu.com → 能解析 → DNS 修复成功
  sudo apt install docker.io → 安装成功
```

---

### 第二步：Docker 拉取镜像失败排查

```
操作：docker pull nginx
报错：failed to resolve reference "docker.io/library/nginx:latest": 
      failed to do request: Head "https://registry-1.docker.io/...": 
      dial tcp: lookup registry-1.docker.io on 127.0.0.53:53: server misbehaving

分析：
  Docker 报错中仍然显示 on 127.0.0.53
  系统 DNS 修好了，但 Docker 有自己独立的 DNS 配置
  Docker 没有配置 daemon.json，默认使用系统的 127.0.0.53

排查命令：
  cat /etc/docker/daemon.json → 文件不存在

根因：
  Docker 使用独立的 DNS 配置
  daemon.json 不存在时默认使用系统的 127.0.0.53
  系统 DNS 通过 Netplan 修复了，但 systemd-resolved 仍然监听 127.0.0.53
  Docker 连接 127.0.0.53 时仍然失败
```

**处理：**

```json
# 创建 Docker 配置文件
sudo tee /etc/docker/daemon.json <<'EOF'
{
    "dns": ["223.5.5.5", "8.8.8.8"],
    "registry-mirrors": [
        "https://docker.1ms.run",
        "https://docker.xuanyuan.me"
    ]
}
EOF

# 重启 Docker
sudo systemctl restart docker
```

```
结果：
  docker pull nginx → 不再报 DNS 错误
  但出现新问题：not found
```

---

### 第三步：Docker 镜像源 not found 排查

```
操作：docker pull nginx
报错：docker.io/library/nginx:latest: not found

分析：
  DNS 解析正常了，但镜像源返回 not found
  镜像源可能失效或不稳定

排查：
  换了多个镜像源均出现类似问题

根因：
  部分国内 Docker 镜像源已失效或不稳定
```

**处理：**

```
多次更换镜像源尝试：
  docker.1ms.run       → not found
  docker.xuanyuan.me   → not found
  registry.cn-hangzhou.aliyuncs.com → 不可用
```

---

### 第四步：IPv6 DNS 污染排查

```
操作：ping docker.io
现象：解析到 2a03:2880:f11c:8083:face:b00c:0:25de

分析：
  这是 Facebook 的 IPv6 地址
  docker.io 被解析到了错误的地址
  说明 IPv6 环境下存在 DNS 污染

排查：
  ping docker.io → 返回 IPv6 地址（2a03:2880:...）
  curl -I https://registry-1.docker.io → 连接超时

根因：
  虚拟机启用了 IPv6
  IPv6 DNS 解析被运营商或网络设备污染
  导致 docker.io 解析到了错误的地址
```

**处理：**

```bash
# 禁用 IPv6
sudo vim /etc/sysctl.conf

# 添加以下内容
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1

# 生效
sudo sysctl -p
```

```
结果：
  ping docker.io → 解析到正确的 IPv4 地址 199.96.58.157
  docker pull nginx → 下载成功

  Status: Downloaded newer image for nginx:latest
  docker.io/library/nginx:latest
```

---

## 根因分析

```
本次故障共涉及 4 个问题，层层叠加：

问题 1：系统 DNS 不可用
  原因：systemd-resolved 本地解析器未正常工作
  影响：apt、ping、curl 等所有依赖 DNS 的命令均失败
  修复：Netplan 配置公共 DNS 223.5.5.5

问题 2：Docker DNS 独立于系统
  原因：Docker 有自己的 DNS 配置（daemon.json）
        未配置时默认使用系统的 127.0.0.53
        虽然系统 DNS 修好了，Docker 仍用旧的 127.0.0.53
  影响：Docker 无法解析任何域名
  修复：daemon.json 中配置 dns: ["223.5.5.5", "8.8.8.8"]

问题 3：国内 Docker 镜像源失效
  原因：多个国内镜像源服务不稳定或已下线
  影响：拉取镜像报 not found 或超时
  修复：更换可用的镜像源

问题 4：IPv6 DNS 污染
  原因：虚拟机启用 IPv6，IPv6 DNS 解析被污染
        docker.io 被错误解析到 Facebook 的 IPv6 地址
  影响：Docker 拉取镜像超时或连接错误地址
  修复：禁用 IPv6
```

---

## 解决方案汇总

```
1. 系统 DNS
   文件：/etc/netplan/00-installer-config.yaml
   修改：添加 nameservers: addresses: [223.5.5.5, 8.8.8.8]
   命令：sudo netplan apply

2. Docker DNS 和镜像源
   文件：/etc/docker/daemon.json
   修改：配置 dns 和 registry-mirrors
   命令：sudo systemctl restart docker

3. 禁用 IPv6
   文件：/etc/sysctl.conf
   修改：添加 disable_ipv6 = 1
   命令：sudo sysctl -p
```

---

## 最终配置文件

### /etc/netplan/00-installer-config.yaml

```yaml
network:
  ethernets:
    ens33:
      dhcp4: true
      nameservers:
        addresses:
          - 223.5.5.5
          - 8.8.8.8
  version: 2
```

### /etc/docker/daemon.json

```json
{
    "dns": ["223.5.5.5", "8.8.8.8"],
    "registry-mirrors": [
        "https://dockerpull.org",
        "https://docker.1ms.run"
    ]
}
```

### /etc/sysctl.conf（追加部分）

```
net.ipv6.conf.all.disable_ipv6 = 1
net.ipv6.conf.default.disable_ipv6 = 1
net.ipv6.conf.lo.disable_ipv6 = 1
```

---

## 经验总结

```
1. DNS 问题要分层排查
   系统 DNS 和应用 DNS 是独立的
   修好一个不代表另一个也好了
   排查时用 ping 域名 和 nslookup 域名 验证

2. IPv6 可能导致意外问题
   虚拟机环境建议禁用 IPv6
   IPv6 DNS 污染在国内网络环境中较常见

3. Docker 镜像源要随时准备更换
   国内镜像源不稳定是常态
   daemon.json 中配置多个镜像源备用
   DNS 也要单独配置，不要依赖系统

4. 排查顺序很重要
   先确认基础网络（ping IP）
   再确认 DNS（ping 域名）
   最后确认应用层（Docker、apt）
   从底层往上逐层排查，效率最高
```

---

## 排查时间线

```
18:00  apt install 失败 → 发现系统 DNS 问题
18:10  修复系统 DNS（Netplan 配置 223.5.5.5）
18:15  apt install 成功，安装 Docker
18:20  docker pull nginx 卡住 → 发现 Docker DNS 问题
18:30  创建 daemon.json，配置 DNS
18:35  docker pull nginx 报 not found → 发现镜像源问题
18:40  多次更换镜像源
18:50  发现 IPv6 DNS 污染问题
18:55  禁用 IPv6
19:00  docker pull nginx 成功

总计耗时：约 1 小时
```

---

这就是一份完整的排错日志。

我搭建 Docker 环境时遇到了 DNS 相关的连锁故障。先是系统 DNS 不可用，修复后发现 Docker 有独立的 DNS 配置，配好后又发现 IPv6 DNS 污染导致解析错误，最终通过禁用 IPv6 解决。整个过程体现了分层排查的思路，从系统层到应用层逐层定位。

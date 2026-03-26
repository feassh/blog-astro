---
title: OCI 多 IPv6 绑定后，Python 请求如何指定出口 IP？
pubDate: 2026-03-26
categories: ['OCI', 'IPv6', 'Python', 'Docker']
description: 在 Oracle Cloud 实例上绑定多个 IPv6 后，如何在 host 网络模式的 Docker 容器里用 Python requests 灵活控制出口 IP，并避免 OCI 公网 IPv4 映射带来的常见误区。
---

# OCI 多 IPv6 绑定后，Python 请求如何指定出口 IP？

最近在折腾 Oracle Cloud 的 IPv6，发现它对多 IPv6 的支持比很多云厂商都大方——单实例可以绑定几十个地址。这让我想到一个很常见的需求：在一台机器上，让不同的 Python 请求走不同的 IPv6 出口。正好项目是用 Docker 跑的，网络模式又是 `host`，于是有了今天这篇实践记录。

## OCI 的多 IPv6 绑定

先说 OCI 这边。Oracle Cloud 对 IPv6 的支持确实挺宽松的：

- 单个实例（准确说是 VNIC 网卡）最多可以绑定 **约 32 个 IPv6 地址**。
- 分配模式是经典的“前缀 + 多地址”：VCN 分配 `/56` 段，子网切出 `/64`，然后在实例上手动或自动分配多个 IPv6。

实际操作中，控制台里可以反复点击“Assign IPv6 Address”来添加，也支持手动指定尾号，这对需要管理多个固定 IP 的场景很友好。

## 多个 IPv6 出站，默认走哪个？

这是个关键问题。绑了一堆 IPv6，系统出站流量会怎么走？

默认行为由 Linux 内核的源地址选择算法（RFC 6724）决定。通常情况下，它会优先使用 **主 IPv6（Primary IPv6）**，也就是第一个分配给网卡的地址。这意味着，如果你什么都不做，所有出站流量都会从同一个 IPv6 出去，其他地址只是静静地躺在网卡上，并不会自动轮换或负载均衡。

要想让不同的请求走不同的 IPv6，必须自己动手控制。

## Docker host 网络模式下的情况

我的项目跑在 Docker 容器里，用了 `--network host`。这种模式下，容器直接共享宿主机的网络命名空间，没有自己的独立 IP 和网络栈隔离。所以容器里的 Python 进程发起网络请求的行为，和直接跑在宿主机上几乎一样。

换句话说，只要某个 IPv6 地址已经绑定在宿主机的网卡上，容器里的 Python 代码理论上就能使用它作为出口源地址。

## Python requests 指定源 IP

`requests` 库本身没有 `source_ip` 这种直接参数，但它提供了一个很优雅的扩展方式：自定义 `HTTPAdapter`。

核心思路是：继承 `HTTPAdapter`，在 `init_poolmanager` 方法里把 `source_address` 参数传递给底层的 `urllib3` 连接池。`source_address` 接受一个元组 `(ip, port)`，端口填 0 表示让系统自动选择临时端口。

下面是我在项目里用的代码：

```python
import requests
from requests.adapters import HTTPAdapter

class SourceIPAdapter(HTTPAdapter):
    def __init__(self, source_ip: str, **kwargs):
        self._source_ip = source_ip
        super().__init__(**kwargs)

    def init_poolmanager(self, connections, maxsize, block=False, **pool_kwargs):
        # 关键：把源地址传给 urllib3 的连接池
        pool_kwargs["source_address"] = (self._source_ip, 0)
        return super().init_poolmanager(connections, maxsize, block=block, **pool_kwargs)

# 用法示例
s = requests.Session()
s.mount("http://", SourceIPAdapter("2001:xxxx::2"))
s.mount("https://", SourceIPAdapter("2001:xxxx::2"))

r = s.get("https://httpbin.org/ip", timeout=10)
print(r.text)
```

对于 IPv6，写法完全一样，直接把 IPv6 地址字符串传进去即可。

## OCI 环境下的特别提醒：公网 IPv4 不是真实地址

如果你的 OCI 实例是带有公网 IPv4 的（比如控制台里显示一个 `158.180.xx.xx` 这样的地址），在配置 `source_address` 时**千万不要直接填这个公网 IPv4**。

原因很简单：OCI 对 IPv4 做的是 NAT 映射，实例网卡上真实存在的 IPv4 是私有地址（例如 `10.0.0.120`）。通过 `ip addr show` 可以看得一清二楚：

```
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> ...
    inet 10.0.0.120/24 metric 100 brd 10.0.0.255 scope global ens3
    inet6 2603:xxxx:f:xxxx:xxxx:xxxx:xxxx:xxxx/128 scope global dynamic noprefixroute
```

系统里只有 `10.0.0.120` 这个私有 IPv4，而公网 IPv4 并没有出现在网卡配置中。如果强行绑定 `158.180.xx.xx`，`requests` 底层会报 `Cannot assign requested address`，因为内核根本找不到这个地址。

所以，在 OCI 上配置 IPv4 出口时，应该使用私有 IPv4：

```python
s = requests.Session()
s.mount("http://", SourceIPAdapter("10.0.0.120"))
s.mount("https://", SourceIPAdapter("10.0.0.120"))
```

这样发起请求后，对外看到的仍然是你的公网 IPv4（因为 OCI 会做源地址转换），但内部你仍然可以通过绑定不同的私有 IPv4（如果分配了多个）来区分出口，只是出口公网 IP 可能还是同一个。

## 提前测试：用 curl 验证出口 IP

在写 Python 代码之前，建议先用 `curl` 确认你想要的源 IP 确实能作为出口。OCI 的网卡上可能同时有多个 IPv4（比如多个私有地址）和多个 IPv6，用 `--interface` 参数就能直接测试：

```bash
curl --interface 10.0.0.120 https://httpbin.org/ip
curl --interface 2603:xxxx:f:xxxx:xxxx:xxxx:xxxx:xxxx https://api64.ipify.org
```

第一条命令会返回你实例的公网 IPv4，第二条会返回你指定的 IPv6 地址。如果某个 IP 绑定失败，`curl` 会直接报错，可以提前发现问题。

## 几个关键注意事项

这套方案跑起来不难，但有几个地方很容易踩坑：

1. **源 IP 必须已绑定在宿主机网卡上**
   用 `ip addr show` 确认要绑定的 IP 确实存在。不要相信云平台控制台展示的“公网 IP”可以直接绑定。

2. **路由必须正确**
   指定源 IP 只告诉内核“用这个地址发起连接”，但流量最终能不能出去，取决于宿主机路由表是否允许这个源地址走对应的出口。如果发现绑定了但连不通，检查一下策略路由。

3. **连接池会复用连接**
   `requests.Session` 会缓存 TCP 连接。同一个 Session 里切换 adapter 并不能真正切换源 IP，因为连接已经建好了。稳妥的做法是：**一个源 IP 对应一个 Session**，按业务逻辑把请求分配到不同的 Session 实例。

4. **如果用了代理**
   当 requests 配置了代理时，源地址绑定可能不会生效，因为实际出口变成了代理服务器。这个场景要单独处理。

## 实际项目中的应用模式

在真实项目里，我一般会这样组织代码：

- 预先定义好需要使用的源 IP 列表（私有 IPv4 或 IPv6 全局地址）
- 为每个源 IP 创建一个独立的 `Session` 实例，并挂载自定义 adapter
- 根据业务（比如不同的下游服务、不同的爬虫任务）选择对应的 Session 发起请求

这样既能保证请求从期望的源 IP 出去，又避免了连接池错乱的问题。

最后记得上线前用 `httpbin.org/ip` 验证一下实际出口 IP，确保配置真正生效。

---
title: 使用 Docker 部署 Gost 转发服务
pubDate: 2024-11-02
categories: ['Docker']
description: ''
---

GOST 全称 “GO Simple Tunnel”，一个使用 Golang 语言实现的安全隧道工具。

假定你已经安装好 Docker，先创建一个 GOST 配置文件，文件名为：“gost.yml”

```yaml
services:
- name: service-0
  addr: :80
  admission: admission-0
  handler:
    type: tcp
  listener:
    type: tcp
  forwarder:
    nodes:
    - name: target-0
      addr: 1.1.1.1:80
- name: service-1
  addr: :443
  admission: admission-0
  handler:
    type: tcp
  listener:
    type: tcp
  forwarder:
    nodes:
      - name: target-1
        addr: 1.1.1.1:443

admissions:
- name: admission-0
  whitelist: true
  matchers:
  - woc.cool
```

该配置文件的作用是将本机的 “80” 和 “443” 端口转发到 Cloudflare 服务器对应的端口上，同时使用 admissions 进行域名白名单鉴权，防止转发服务被滥用。

然后创建一个 docker-compose.yml 配置文件，用于启动 Docker 服务。

```yaml
version: '3.9'

services:
  gost:
    container_name: gost
    image: gogost/gost:版本号
    # 端口映射
    ports:
      - 80:80
      - 443:443
    volumes:
      # 将当前目录的 gost.yml 文件映射到 docker 容器的指定目录中
      - ./gost.yml:/etc/gost/gost.yml
    restart: "always"
```

上面的 Docker 配置文件默认使用 “bridge” 桥接的网络模式，所以需要进行端口映射。但是对于 GOST 我更倾向于使用 “host” 模式，因为 “bridge” 模式对于需要高性能网络的应用来说是会有一些性能损耗。下面是使用 “host” 的配置文件。

```yaml
version: '3.9'

services:
  gost:
    container_name: gost
    image: gogost/gost:版本号
    volumes:
      # 将当前目录的 gost.yml 文件映射到 docker 容器的指定目录中
      - ./gost.yml:/etc/gost/gost.yml
    restart: "always"
    # 使用 host 模式，不需要进行端口转发，docker 容器内部使用的端口会直接作用在宿主机上
    network_mode: host
```

最后开放服务器防火墙对应的端口，并使用相应的 docker compose 命令进行操作即可。

```shell
docker compose up -d # 下载镜像并启动容器
docker compose down -v # 停止容器
```
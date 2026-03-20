---
title: 用 Nginx 的 stream 模块做基于 SNI 的端口转发
pubDate: 2026-03-20
categories: ['Nginx', '网络']
description: 记录一种利用 Nginx stream 模块和 ssl_preread 指令，根据访问域名（SNI）进行 TCP 端口转发的配置方法，适合在四层代理场景下复用同一个 443 端口。
---

最近在折腾服务器的时候遇到一个场景：手头只有一个公网 IP，但有多台后端服务器需要分别提供 HTTPS 服务。想让用户通过不同域名访问时，自动转发到对应的后端服务器，但又不想在每个后端服务器上都单独绑定公网 IP，也不想用七层的反向代理来解密 HTTPS（那样证书配置和性能开销都比较麻烦）。

后来发现 Nginx 的 `stream` 模块配合 `ssl_preread` 可以很优雅地解决这个问题。简单来说就是在四层（TCP）层面根据 TLS 握手时的 SNI（Server Name Indication）信息来做路由转发。下面记录一下具体的配置方法和一些需要注意的点。

## 核心配置

我最终的配置大概是这样的：

```nginx
events {
    # 即使为空，也必须保留 events 块
}

stream {
    map $ssl_preread_server_name $backend {
        example.com      104.19.143.56:443;
        api.example.com  104.19.143.56:443;
        # default        1.1.1.1:443;  # 默认后端，处理未匹配的域名
    }

    server {
        listen 443;
        proxy_pass $backend;
        ssl_preread on;
    }
}
```

## 关键点解读

### 1. `events` 块不能省略

即使里面什么都不写，`events {}` 这个块也必须有。这是 Nginx 的语法要求，少了它启动时会报错。

### 2. `stream` 上下文

`stream` 是 Nginx 用来处理 TCP/UDP 流量的模块，和常见的 `http` 是平级的。它工作在四层，不会解析 HTTP 协议的内容，所以效率比七层代理高，很适合做端口转发。

### 3. `ssl_preread` 的作用

`ssl_preread on;` 这行是关键。它会让 Nginx 在建立 TCP 连接后，先 peek（偷看）一下客户端发来的 TLS ClientHello 数据包，从中读取 SNI 字段（即客户端请求的域名），但**不进行解密**。读取完 SNI 后，Nginx 再根据 `map` 里定义好的规则，将整个 TCP 连接透明地转发给后端。

### 4. 动态选择后端

`map $ssl_preread_server_name $backend` 这行定义了一个变量 `$backend`，它的值会根据 `$ssl_preread_server_name`（SNI 域名）来变化。这样就可以实现不同域名指向不同的后端地址。

如果访问的域名不在列表中，可以用 `default` 指定一个兜底的后端，避免连接被拒绝。

## 一些实际操作时的注意事项

- **编译 Nginx 时需包含 stream 模块**
  大多数发行版的官方 Nginx 包默认是包含了 `stream` 模块的，但如果你是自己编译安装的，需要加上 `--with-stream` 和 `--with-stream_ssl_preread_module` 这两个编译选项。

- **防火墙和安全组**
  这种配置下，Nginx 监听在公网的 443 端口，后端服务器上只需要监听内网端口，不需要开放给公网。记得在防火墙或安全组里放行 Nginx 所在服务器的 443 端口。

- **后端服务器的证书**
  由于 Nginx 只是做四层转发，并不解密流量，所以 SSL 证书的验证仍然由后端服务器完成。也就是说，每个后端服务器需要配置好对应域名的证书，否则客户端会收到证书错误的提示。

- **性能考虑**
  `ssl_preread` 只读取 TLS ClientHello 的一小部分数据，开销很小。相比用 `http` 模块做反向代理（需要处理 TLS 握手、解密、再重新加密），这种四层方案资源占用更低，延迟也更小。

## 收尾

这种方案比较适合资源有限、需要在单一 IP 上暴露多个 HTTPS 服务的场景。比如在一台低配 VPS 上统一接收流量，再根据域名分发给内网的多台机器。配置起来也不复杂，只要理清 `stream` 和 `map` 的逻辑，再注意一下 `events` 块的存在，基本就能跑起来。

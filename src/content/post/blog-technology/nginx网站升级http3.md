---
title: nginx网站升级http3
publishDate: 2025-10-28 23:06:18
tags: [nginx]
description: nginx网站升级http3
---

## HTTP2 的网站配置

```nginx
server {
    # 监听IPv4 443端口，并使用SSL/TLS(HTTPS)协议进行加密通信
    listen                     443 ssl;
    # 同上，只是增加了IPv6
    listen                     [::]:443 ssl;
    # 启用HTTP2，注意在1.25.1版本后，启用HTTP2不需要在listen行中增加http2参数，可以作为单独的指令使用
    http2                      on;

    # 规定SSL协议版本，禁用SSL v3.0，TLS v1.0 及 v1.1
    ssl_protocols              TLSv1.2 TLSv1.3;
    # 规定TLS加密算法，排除弱加密算法
    ssl_ciphers TLS13_AES_128_GCM_SHA256:TLS13_AES_256_GCM_SHA384:TLS13_CHACHA20_POLY1305_SHA256:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305;
    # 优先使用服务器端加密算法，而不是客户端偏好，保证安全性与算法性能
    ssl_prefer_server_ciphers  on;

    # 全链证书文件路径
    sl_certificate             /path/to/fullchain.crt;
    # 证书相匹配的私钥文件路径
    ssl_certificate_key        /path/to/private.key;

    # 该Server配置响应的域名，如匹配上，则使用此Server块配置
    server_name                host.example.com;

    # “/”表示匹配所有访问该Server下的URL
    location / {
        # 网站根目录路径
        root                   /path/to/website
    }
}
```

<!-- more -->

## HTTP3 的网站配置

在 HTTP2 配置不变的基础上，再增加一些配置即可。

```nginx
server {
    # HTTP2配置不变，这里省略不写
    # 注意HTTP3底层不再是TCP协议而是UDP协议，因此不存在端口竞争，不是必需在listen中使用reuseport参数，
    # 但使用了后允许多个Worker共享端口，显著提高并发性能与负载均衡效率
    # 在IPv4 443上监听QUIC（HTTP3）协议
    listen                     443 quic reuseport;
    # 同上在IPv6上监听
    listen                     [::]:443 quic reuseport;
    # 启用HTTP3
    http3                      on;
    # 启用 QUIC 地址验证功能。这包括在重试数据包或 NEW_TOKEN 帧中发送新令牌，以及验证在初始数据包中收到的令牌。
    quic_retry                 on;
    # 启用分段卸载以优化的批处理模式启用发送。
    # GSO(Generic Segmentation Offload)是一种内核特性，允许应用程序（Nginx）将大的数据块一次性提交给内核，由内核在发送时进行分段。
    # 这减少了内核与用户空间之间的交互，并减轻了 CPU 负载，从而提高 QUIC 性能。
    quic_gso                   on;
    # 增加协议切换告知头： Nginx 在响应客户端（通过 TCP/HTTP/2 连接）时，返回这个 HTTP 头部。
    # 它告诉兼容的客户端：“我在这个端口上（:443）也支持 HTTP/3 (h3)，有效期是 86400 秒。”
    # 客户端收到后，会尝试切换到更快的 QUIC/HTTP/3 连接。
    add_header Alt-Svc 'h3=":443"; ma=86400';
    # 增加 启用 HSTS (HTTP严格传输安全) 头：
    # Strict-Transport-Security 头部要求浏览器在接下来的 31536000 秒（一年）内，只能通过 HTTPS 访问该网站，即使输入的是 http://。
    # includeSubDomains 适用于所有子域名，preload 表示希望被加入浏览器内置的 HSTS 列表中（需要单独提交）。
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"

    # 启用 TLS 0-RTT (零往返时间)：
    # 这是 TLS 1.3 的一个功能，允许客户端在第一个连接数据包中发送应用数据（在 QUIC 中体现为 Initial 数据包），从而显著加快后续访问的加载速度。
    ssl_early_data             on;
}
```

## 抗量子

量子计算机的出现，对现在的加密算法是一个威胁，现在不可被破解的加密，在量子计算机上则可以被破解。TLS 证书加密的 HTTPS 连接在量子计算机环境下也变得不再安全。攻击者可以先存储现在的加密流量，等若干年后，量子计算机问世，再进行解密存储的流量。要解决这个问题，一方面是在 TLS 证书上改用抗量子的算法，但现在各大证书商尚未支持。现阶段的折中做法是混合加密，在 X25519 等经典算法基础上混合抗量子算法，即使未来量子计算机攻破了经典算法，但信息是经过双重混合加密的，抗量子算法无法被攻破，则信息仍然是安全的。

在 Nginx 中配置抗量子加密，注意服务端的 Nginx 与 OpenSSL 版本，抗量子只在 OpenSSL 大于 3.5 的版本中可用。

```nginx
server {
    # 其他TLS配置不变，只需要增加一行
    # 定义 Nginx 在 TLS 握手过程中，用于进行密钥交换的 椭圆曲线列表，按偏好顺序排列。
    # X25519MLKEM768 混合后量子算法 将经典的 X25519（高性能椭圆曲线）与 ML-KEM-768（基于格的后量子算法）结合
    # X25519 现代椭圆曲线 它比传统曲线（如 prime256v1）更快、更安全、实现更简单，是目前最受欢迎的选择
    # prime256v1 传统 NIST 曲线 这是美国国家标准与技术研究院 (NIST) 推荐的曲线之一，也称为 P-256
    # secp384r1 高强度 NIST 曲线 也称为 P-384，比 P-256 提供更高的安全级别，但计算开销也更大
    ssl_ecdh_curve             X25519MLKEM768:X25519:prime256v1:secp384r1;
}
```

---
title: "使用 Caddy 在非 443 端口开启 HTTPS"
date: 2020-10-17T21:18:29+08:00
---

最近新入手了 HPE ProLiant MicroServer Gen10 Plus ，拿来做家庭服务器，以期避免长期租用云服务器。 虽然家庭服务器对比云服务器在性价比上有很明显的优势，但是却缺少了云服务的一项重要内容： 固定公网 IP 。 平时我们使用公网 IP 是为了可以随时随地访问服务器，而这个需求通过 DDNS 也可以实现，只是访问需要用域名而不是裸 IP 。

通常 DDNS 由路由器来完成，并通过端口映射的方式把内网的服务暴露在外。 国内要求所有 IP 只有在域名备案之后才可以开放 80 和 443 端口对外提供 Web 服务，云服务器可以照常备案，而普通家庭宽带用户被分配的公网 IP （某些运营商/地区甚至不会分配公网 IP）是每隔一段时间就会发生变化的，因此没有办法走常规备案的流程。 因此在国内只能通过其它端口来访问家里的 Web 应用，但是这样会导致部署自动 HTTPS （通过Let's Encrypt）变得很麻烦，本文会介绍如何通过 Caddy 在非 443 端口部署 HTTPS 服务以及背后的原理。

# Let's Encrypt 与 Challenge

我们需要从证书颁发机构获取证书来在网站上启用 HTTPS ， Let's Encrypt 是一个免费此类颁发证书的机构，以下内容摘自 [Let's Encrypt/快速入门](https://letsencrypt.org/zh-cn/getting-started/) 与 [Let's Encrypt/验证方式](https://letsencrypt.org/zh-cn/docs/challenge-types/) ：

> 为了在您的网站上启用 HTTPS，您需要从证书颁发机构（CA）获取证书（一种文件）。Let’s Encrypt 是一个证书颁发机构（CA）。要从 Let’s Encrypt 获取您网站域名的证书，您必须证明您对域名的实际控制权。您可以在您的 Web 主机上运行使用 ACME 协议的软件来获取 Let’s Encrypt 证书。
>
> 当您从 Let’s Encrypt 获得证书时，我们的服务器会验证您是否使用 ACME 标准定义的验证方式来验证您对证书中域名的控制权。大多数情况下，验证由 ACME 客户端自动处理，但如果您需要做出一些更复杂的配置决策，那么了解更多有关它们的信息会很有用。

通过 ACME 协议向 Let's Encrypt 证明自己的域名所有权的过程就叫做 Challenge （验证），目前有三种 Challenge 的方式：

* HTTP-01
* DNS-01
* TLS-SNI-01 （已禁用）
* TLS-ALPN-01

HTTP-01 是目前最常见的验证方式，但是该验证方式需要通过 80 端口开放一个路径给 Let's Encrypt 访问它提供的 token 来验证你的域名所有权，因此在 80 端口被封锁的情况下这个验证方式是不现实的。 类似的， TLS-ALPN-01 需要通过 443 端口访问来验证，也是行不通。 这样对于国内家庭带宽用户来说就只剩下了一种方式： DNS-01 。

> 此验证方式要求您在该域名下的 TXT 记录中放置特定值来证明您控制域名的 DNS 系统。该配置比 HTTP-01 略困难，但可以在某些 HTTP-01 不可用的情况下工作。它还允许您颁发通配符证书。在 Let’s Encrypt 为您的 ACME 客户端提供令牌后，您的客户端将创建从该令牌和您的帐户密钥派生的 TXT 记录，并将该记录放在 _acme-challenge.<YOUR_DOMAIN> 下。然后 Let’s Encrypt 将向 DNS 系统查询该记录。如果找到匹配项，您就可以继续颁发证书！
>
> 由于颁发和续期的自动化非常重要，只有当您的 DNS 提供商拥有可用于自动更新的 API 时，使用 DNS-01 验证方式才有意义。我们的社区在此处提供了此类 DNS 提供商的列表。您的 DNS 提供商可能与您的域名注册商（您从中购买域名的公司）相同或不同。如果您想更改 DNS 提供商，只需在注册商处进行一些小的更改，而无需等待域名即将到期。

这种方式是通过服务器修改 DNS 记录， Let's Encrypt 去查询的方式来验证域名所有权，因此需要服务器有权通过 API 访问 DNS 提供商并修改记录。

# Caddy 配置

目前常见的服务器开启 HTTPS 并自动续期的方案有两种： Caddy 和 Nginx + Certbot 。 后者配置的复杂程度很高，就算利用 docker 也不能避免冗长的配置编写过程，而 Caddy 内置了 HTTPS 以及自动续期的支持，配置也相对简洁，因此下面介绍使用 Caddy 部署 HTTPS 的方法。

Caddy 默认使用 HTTP-01 验证方式，要通过 DNS-01 验证需要对应的插件支持，通过软件源安装的 Caddy 没有这些插件，需要在[下载页](https://caddyserver.com/download)下载或是通过 xcaddy 从源码构建。

下载页 caddy-dns 开头的几个插件就是 DNS-01 验证的插件，插件与 DNS 提供商有关，因此有好几个，本文使用阿里云 DNS ，因此选中了 caddy-dns/lego-deprecated 插件（这个插件最近在用 go 重写，新版功能还不全，因此先用着旧版）。 Linux amd64 + lego 插件的二进制包下载地址为 https://caddyserver.com/api/download?os=linux&arch=amd64&p=github.com%2Fcaddy-dns%2Flego-deprecated&idempotency=80128464204175 。

下载之后执行 `chmod +x caddy && mv caddy /usr/bin` 赋予执行权限并移到 bin 目录。 下面编写 caddy 的配置文件 Caddyfile ：

```text
{
  http_port 1999
  https_port 2000
}

xxx.yourdomain.com {
  reverse_proxy localhost:2001

  tls {
    dns lego_deprecated alidns
  }
}
```

这里第一个大括号内是全局配置，声明了 caddy 要在哪个端口监听外部的 HTTP / HTTPS 请求，这个端口也是做 DDNS 的路由器需要转发到外部的端口。 后面 xxx.yourdomain.com 需要替换成个人的域名，大括号内是这个域名的配置： 反向代理本地 2001 端口的 Web 服务，且 tls 证书申请通过 dns 方式进行，使用的插件为 lego_deprecated ， alidns 为[插件的参数](https://github.com/caddy-dns/lego-deprecated)。

根据 [lego 的文档](https://go-acme.github.io/lego/dns/alidns/) ，我们需要把阿里云的 Access key ID 与 Access Key secret 设置为环境变量，这两个值可以在阿里云的控制台获取。 鼠标悬停在右上角头像上，弹出的名片里有一项 Access Key 管理，进入之后点击创建 Access Key 就可以得到一组 Access key ID 与 Access Key secret ，将这两个值保存在一个环境变量文件 `alidns.env` 里：

```text
ALICLOUD_ACCESS_KEY=id
ALICLOUD_SECRET_KEY=secret
```

作为测试，可以先起一个 python 简易服务器：

```bash
python3 -m http.server 2001
```

之后运行 caddy 服务：

```bash
caddy run --config Caddyfile --envfile alidns.env
```

如果一切正常，应该可以在终端日志看到 caddy 通过 dns challenge 申请到证书的过程，之后就可以通过域名 + 非 443 端口通过 HTTPS 访问本地的 Web 服务了。

# 合规性提醒

目前对于个人用户能否搭建服务器貌似没有明确的规定（无论是否为80/443端口），此前也有非标准端口搭建 Web 服务导致宽带被封的[案例](https://www.v2ex.com/t/608821?p=1)，因此建议不要把此方案应用于对流量与稳定性有较高需求的场景。

---
title: "Use Caddy to enable HTTPS on ports other than 443"
date: 2020-10-17T21:18:29+08:00
---

__Note: This article is automatically translated, please turn to the Chinese version for more accurate expression if possible.__

Recently I started with HPE ProLiant MicroServer Gen10 Plus and used it as a home server to avoid long-term rental of cloud servers. Although home servers have obvious advantages in cost performance compared with cloud servers, they lack an important content of cloud services: fixed public IP. We usually use the public IP in order to be able to access the server anytime and anywhere, and this requirement can also be achieved through DDNS, but access requires a domain name instead of a bare IP.

Usually DDNS is completed by routers and exposes intranet services to the outside through port mapping. Domestically, all IPs are required to open ports 80 and 443 to provide external web services only after the domain name has been filed. Cloud servers can be filed as usual, while ordinary household broadband users are assigned public IPs (some operators/regions may not even assign public IPs). Internet IP) changes every once in a while, so there is no way to go through the regular filing procedures. Therefore, you can only access home web applications through other ports in China, but this will cause the deployment of automatic HTTPS (via Let's Encrypt) to become very troublesome. This article will introduce how to deploy HTTPS services on non-443 ports through Caddy and the principles behind them.

# Let's Encrypt and Challenge

We need to obtain a certificate from a certificate authority to enable HTTPS on the website. Let's Encrypt is a free certificate issuing authority. The following content is taken from[Let's Encrypt/Quick Start](https://letsencrypt.org/zh-cn/getting-started/) versus [Let's Encrypt/Verification Method](https://letsencrypt.org/zh-cn/docs/challenge-types/) ：

> In order to enable HTTPS on your website, you need to obtain a certificate (a file) from a certificate authority (CA). Let’s Encrypt is a certification authority (CA). To obtain a certificate for your website domain name from Let’s Encrypt, you must prove your actual control of the domain name. You can run software that uses the ACME protocol on your web host to obtain the Let’s Encrypt certificate.
>
> When you get a certificate from Let’s Encrypt, our server will verify that you use the verification method defined by the ACME standard to verify your control of the domain name in the certificate. In most cases, verification is handled automatically by the ACME client, but if you need to make some more complex configuration decisions, it is useful to learn more about them.

The process of proving your own domain name ownership to Let's Encrypt through the ACME protocol is called Challenge (verification). There are currently three Challenge methods:

* HTTP-01
* DNS-01
* TLS-SNI-01 (disabled)
* TLS-ALPN-01

HTTP-01 is currently the most common verification method, but this verification method needs to open a path through port 80 for Let's Encrypt to access the token provided by it to verify your domain name ownership, so this verification method is when port 80 is blocked Unrealistic. Similarly, TLS-ALPN-01 needs to be accessed through port 443 for verification, which is also unworkable. In this way, there is only one way left for domestic household bandwidth users: DNS-01.

> This verification method requires you to put a specific value in the TXT record under the domain name to prove that you control the DNS system of the domain name. This configuration is slightly more difficult than HTTP-01, but it can work in some cases where HTTP-01 is not available. It also allows you to issue wildcard certificates. After Let's Encrypt provides your ACME client with a token, your client will create a TXT record derived from the token and your account key, and place the record under _acme-challenge.<YOUR_DOMAIN> . Let’s Encrypt will then query the DNS system for the record. If a match is found, you can continue to issue certificates!
>
> Since the automation of issuance and renewal is very important, it only makes sense to use the DNS-01 authentication method if your DNS provider has an API that can be used for automatic updates. Our community provides a list of such DNS providers here. Your DNS provider may be the same as or different from your domain registrar (the company from which you purchased the domain name). If you want to change your DNS provider, just make some small changes at the registrar without waiting for the domain name to expire soon.

This method uses the server to modify the DNS records and Let's Encrypt to query to verify the ownership of the domain name. Therefore, the server is required to have the right to access the DNS provider through the API and modify the records.

# Caddy configuration

At present, there are two common server open HTTPS and automatic renewal schemes: Caddy and Nginx + Certbot. The latter configuration is very complicated. Even if you use docker, you can't avoid the lengthy configuration writing process. Caddy has built-in HTTPS and automatic renewal support, and the configuration is relatively simple. Therefore, the following describes the method of using Caddy to deploy HTTPS.

Caddy uses the HTTP-01 authentication method by default. To pass DNS-01 authentication, the corresponding plug-in support is required. Caddy installed through the software source does not have these plug-ins.[Download page](https://caddyserver.com/download)Download or build from source via xcaddy.

The plug-ins at the beginning of the download page caddy-dns are the plug-ins for DNS-01 verification. The plug-ins are related to the DNS provider, so there are several. This article uses Alibaba Cloud DNS, so the caddy-dns/lego-deprecated plug-in is selected (this plug-in Recently rewritten with go, the new version is not fully functional, so the old version is used first). The binary package download address of the Linux amd64 + lego plug-in is https://caddyserver.com/api/download?os=linux&arch=amd64&p=github.com%2Fcaddy-dns%2Flego-deprecated&idempotency=80128464204175.

Execute after download`chmod +x caddy && mv caddy /usr/bin` Give execution permission and move to the bin directory. Write the configuration file Caddyfile of caddy below:

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

The first brace here is the global configuration, which declares which port caddy will listen to external HTTP/HTTPS requests. This port is also the port that the DDNS router needs to forward to the outside. The following xxx.yourdomain.com needs to be replaced with a personal domain name. The configuration of this domain name is in braces: Reverse proxy the web service of the local 2001 port, and the tls certificate application is done through dns, the plug-in used is lego_deprecated and alidns[Plug-in parameters](https://github.com/caddy-dns/lego-deprecated)。

According to[lego's documentation](https://go-acme.github.io/lego/dns/alidns/) , We need to set Alibaba Cloud's Access key ID and Access Key secret as environment variables. These two values can be obtained on the Alibaba Cloud console. Hover the mouse on the avatar in the upper right corner, and there is an Access Key management in the business card that pops up. After entering, click Create Access Key to get a set of Access key ID and Access Key secret, and save these two values in an environment variable file`alidns.env` in:

```text
ALICLOUD_ACCESS_KEY=id
ALICLOUD_SECRET_KEY=secret
```

As a test, you can start a simple python server:

```bash
python3 -m http.server 2001
```

Then run the caddy service:

```bash
caddy run --config Caddyfile --envfile alidns.env
```

If everything is normal, you should be able to see the process of caddy applying for a certificate through dns challenge in the terminal log, and then you can access the local web service through HTTPS through the domain name + non-443 port.

# Compliance reminder

At present, it seems that there is no clear regulation on whether individual users can build a server (regardless of port 80/443). Previously, there were also non-standard ports to build web services that caused broadband to be blocked.[Case study](https://www.v2ex.com/t/608821?p=1)Therefore, it is recommended not to apply this solution to scenarios with high requirements for traffic and stability.

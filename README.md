# telegram

最近有使用 Telegram 机器人相关的服务，之前是部署在境外服务器上，运行良好。某天脑子抽风把服务迁移到国内机器了，发现一直 Timeout，后来才反应过来 Telegram 被墙了。由于我使用的是 Telegram bot 相关服务，请求的都是 https://api.telegram.org 这个域名的地址，于是我就想到了能不能借助国外的机器设置反向代理解决这个问题。

##中间人劫持

首先我想到的是不修改服务代码，直接将我本地的 api.telegram.org 解析到我的境外服务器，然后在服务器上做反向代理。这个方案是可行的，唯一需要解决的是服务器上 api.telegram.org 证书信任的问题。解决方法也很简单，服务器上自建证书后在本地设置强制信任即可。

这里比较坑的一点在于网上给的教程一般都是说如何在 Linux 上创建自建证书然后在客户端例如 Windows 上信任。很少有教程会说如何在 Linux 上信任的，所以当时搜的资料很少。使用如下命令可为 Linux 添加证书：

sudo mkdir /usr/share/ca-certificates/extra
sudo cp foo.crt /usr/share/ca-certificates/extra/foo.crt
sudo dpkg-reconfigure ca-certificates

##反向代理

由于我这边 Docker 环境原因发现中间人劫持行不通。后来就想了个退而求其次的方法，还是反向代理 api.telegram.org，但是用自己的域名 telegram.imnerd.org，代码里修改一下调用的接口地址即可。以下是具体的 Nginx 配置：

sever {
  listen 80;
  listen 443 ssl;
  server_name     telegram.imnerd.org;
  location ~* ^/bot {
    proxy_buffering off;
    proxy_pass      https://api.telegram.org$request_uri;
  }
  location ^~ /.well-known/acme-challenge/ {
    alias   /etc/nginx/ssl/challenges/;
    try_files       $uri =404;
  }
  ssl on;
  ssl_certificate /etc/nginx/ssl/chained.pem;
  ssl_certificate_key /etc/nginx/ssl/domain.key;
  ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
  ssl_prefer_server_ciphers       on;
  ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA RC4 !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS";
}

除了 proxy_pass 反代之外，同样还需要处理 https 证书的问题。这里我使用了 Let's encrypt 证书。最后直接使用 curl 测试即可。如果返回 JSON 内容即为正常。

curl https://telegram.imnerd.org/bot

参考资料：

![How do I install a root certificate?]（https://askubuntu.com/questions/73287/how-do-i-install-a-root-certificate）

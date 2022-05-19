---
title: 给博客启用HTTPS证书
date: 2016-04-17 20:08:39
tags:
---

### 开启443端口

```
# iptables -A INPUT -p tcp --dport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
# iptables -A OUTPUT -p tcp --sport 443 -m state --state NEW,ESTABLISHED -j ACCEPT
# service iptables save
```

### 申请证书

生成用户标识私钥和域名证书私钥

```
mkdir -p /data/ssl
cd /data/ssl
openssl genrsa 4096 > account.key
openssl genrsa 4096 > domain.key
```

生成证书请求文件

    openssl req -new -sha256 -key domain.key -subj "/" -reqexts SAN -config <(cat /etc/pki/tls/openssl.cnf <(printf "[SAN]\nsubjectAltName=DNS:ihuhao.com,DNS:www.ihuhao.com,DNS:app.ihuhao.com")) > domain.csr


使用`acme-tiny`脚本到`Let's Encrypt` 申请证书, 下载acme_tiny脚本

    wget https://raw.githubusercontent.com/diafygi/acme-tiny/master/acme_tiny.py

创建一个目录用于存放证书验证过程中的临时文件

    mkdir  /data/www/challenges/

设置nginx, 以`blog.ihuhao.com`为例, 如下

```
server {
    listen 80;
    listen 443 ssl;
    server_name  blog.ihuhao.com;

    ssl_certificate     /data/ssl/chained.pem;
    ssl_certificate_key /data/ssl/domain.key;

    ssl_prefer_server_ciphers on;
    ssl_dhparam /data/ssl/dhparam.pem;
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers "EECDH+ECDSA+AESGCM EECDH+aRSA+AESGCM EECDH+ECDSA+SHA384 EECDH+ECDSA+SHA256 EECDH+aRSA+SHA384 EECDH+aRSA+SHA256 EECDH+aRSA+RC4 EECDH EDH+aRSA !aNULL !eNULL !LOW !3DES !MD5 !EXP !PSK !SRP !DSS !RC4";
    keepalive_timeout 70;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m; 

    add_header Strict-Transport-Security max-age=63072000;
    add_header X-Frame-Options DENY;
    add_header X-Content-Type-Options nosniff;

    location ^~ /.well-known/acme-challenge/ {
        alias /data/www/challenges/;
        autoindex on;
    }

    location / {
        alias /home/wawa/src/onlytiancai.github.io/;
        expires 1h;
        autoindex on;
    }

}

```

执行脚本，申请证书

    python acme_tiny.py --account-key ./account.key --csr ./domain.csr --acme-dir /data/www/challenges/ > ./signed.crt

脚本执行过程中，会在`/data/www/challenges`目录下生成一个随机文件，然后`Let's Encrypt`会访问`http://blog.ihuhao.com/.well-known/acme-challenge/`下的文件用于验证域名，所以上面的nginx配置要配置好，整个脚本才能执行成功。

下载中间证书，并和上一步生成的crt粘合在一起，生成nginx使用的证书。

```
wget -O - https://letsencrypt.org/certs/lets-encrypt-x1-cross-signed.pem > intermediate.pem
cat signed.crt intermediate.pem > chained.pem
```

nginx配置，上面已经贴过了，证书路径，证书私钥路径，证书参数配置等都配好了，刷新下nginx配置，网站就可以用https访问了。

    /usr/local/nginx/sbin/nginx -s reload

### hexo改动

因为上面的nginx配置开启了`HSTS`, 但hexo的yilia模版里有的地方强引用了http的脚本，所以要做适当的修改，目前主要有如下两个路径

    ./themes/yilia/layout/_partial/after-footer.ejs
    ./themes/yilia/layout/_partial/mathjax.ejs

把`http://xxx`改成`//xxx`，然后`hexo generate`生成一下博客就可以了

### 参考链接

- [Let's Encrypt，免费好用的 HTTPS 证书](https://imququ.com/post/letsencrypt-certificate.html)

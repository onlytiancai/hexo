---
title: 开启php-fpm status page
date: 2016-03-17 12:25:00
tags:
    - php
    - 监控
---

要监控PHP运行情况，打开php-fpm的status page可以提供很多有用的信息。

找到php-fpm的配置路径和master的pid

    # ps -ef | grep "php-fpm: master"
    root     31236     1  0 11:46 ?        00:00:00 php-fpm: master process (/usr/local/php/etc/php-fpm.conf)  

修改php-fpm配置，打开status page

    vi /usr/local/php/etc/php-fpm.conf
    pm.status_path = /php_status

重启php-fpm

    kill -USR2 31236

找到nginx配置路径

    # ps -ef | grep "nginx: master"
    root      3813     1  0 Mar09 ?        00:00:00 nginx: master process /usr/local/nginx/sbin/nginx

修改nginx配置，增加localhost的server节

    server  
    {
        listen       80;     
        server_name  localhost;
        error_log logs/localhost-error_log;
        access_log logs/localhost-access_log;

        location ~ ^/php_status$ 
        {       
            allow 127.0.0.1;
            deny all;

            include fastcgi_params;
            fastcgi_pass unix:/var/run/php5-fpm.socket; # 根据真实情况改动, 详见php-fpm配置
            fastcgi_param SCRIPT_FILENAME $fastcgi_script_name;
        }       

    }

重启nginx

    /usr/local/nginx/sbin/nginx -t
    /usr/local/nginx/sbin/nginx -s reload

测试php-fpm状态页

    # curl localhost/php_status           
    pool:                 www
    process manager:      static
    start time:           17/Mar/2016:12:09:30 +0800
    start since:          747
    accepted conn:        4992
    listen queue:         0
    max listen queue:     0
    listen queue len:     0
    idle processes:       196
    active processes:     4
    total processes:      200
    max active processes: 15
    max children reached: 0
    slow requests:        0

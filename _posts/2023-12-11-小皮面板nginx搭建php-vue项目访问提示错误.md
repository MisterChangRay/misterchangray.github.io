---
layout: post
title:  "小皮面板nginx搭建php-vue项目访问提示错误"
date:   2023-12-11 13:29:20 +0800
categories:
      - 杂项
tags:
      - 小皮面板
---

小皮面板对于php搭建开发测试环境太友好了，这里强推一波；

最近接触一个php项目，前端使用的是vue. 
小皮面板nginx+php配置完成后。不能正常访问页面，提示404；

后经多方查找，nginx对于php映射需要单独配置，配置内容如下：
```
server {
        listen        80;
        server_name  ces ces;
        root   "D:/workspace/cesi_cn/public";
        location / {
            index index.php index.html error/index.html;
            error_page 400 /error/400.html;
            error_page 403 /error/403.html;
            error_page 404 /error/404.html;
            error_page 500 /error/500.html;
            error_page 501 /error/501.html;
            error_page 502 /error/502.html;
            error_page 503 /error/503.html;
            error_page 504 /error/504.html;
            error_page 505 /error/505.html;
            error_page 506 /error/506.html;
            error_page 507 /error/507.html;
            error_page 509 /error/509.html;
            error_page 510 /error/510.html;
            try_files $uri $uri/ /index.php?$query_string;
            proxy_read_timeout 180;
            proxy_connect_timeout 180;
            include D:/workspace/cesi_cn/public/nginx.htaccess;
            autoindex  off;
        }
        location ~ \.php(.*)$ {
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_read_timeout 1800;
            fastcgi_split_path_info  ^((?U).+\.php)(/?.+)$;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            fastcgi_param  PATH_INFO  $fastcgi_path_info;
            fastcgi_param  PATH_TRANSLATED  $document_root$fastcgi_path_info;
            include        fastcgi_params;
        }
		access_log E:/phpstudy_pro/Extensions/Nginx1.15.11/logs/ces_acess.log;
}


主要是第1段，在小皮面板`网站->管理->修改->错误页面`编辑框最后新增：`  try_files $uri $uri/ /index.php?$query_string;` 即可

```

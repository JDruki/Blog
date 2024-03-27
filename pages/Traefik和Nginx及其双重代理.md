# 前言
前面介绍了 [[Traefik]]，但是在某些情况下只使用它没法完成一些特殊情况，所以还需要使用 [[Nginx]]来联动使用。
- # 配置
- ## Nginx双重代理
  
  在之前 [[Traefik]] ，可以看到我反代了一个Nginx的镜像，使用的是cafenya.one服务器，这样我们可以编写nginx配置实现双重代理的效果。  
  
  ```json
  server {
  listen         80 default_server;
  server_name    cafenya.one;
  
  # Traefik -> nginx -> synapse
  location /_matrix {
    proxy_pass http://synapse:8008;
    proxy_set_header X-Forwarded-For $remote_addr;
    client_max_body_size 128m;
  }
  
  location /.well-known/matrix/ {
    root /var/www/;
    default_type application/json;
    add_header Access-Control-Allow-Origin  *;
  }
  
  # 反向代理到 web:3000
  location / {
    proxy_pass http://web:3000;
    proxy_set_header X-Forwarded-For $remote_addr;
    client_max_body_size 128m;
  }
  
  location /_wechat/ {
    proxy_pass http://matrix-wechat:20002;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
  }
  }
  ```
  因为是在Traefik下层所以我们不用去向反代的是什么域名，配置证书等，专注与服务即可。  
  这里只是简单的介绍，具体Traefik和Nginx怎么使用我会开专栏。文章包含大部分自己理解，如果有错误可以直接指证出来，互相交流。
- #Docker #代理
-
-
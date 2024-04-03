- [[Feb 8th, 2024]] #Docker #博客
- # 前言 
  在前面的[[Docker基础语法]]中，我详细解释了docker一些比较常用的命令，但是有一块因为篇幅原因并没有进行介绍，就是Docker Network,在我初期学习中，因为不知道Docker网络的运作模式，于是在反代及容器沟通过程中踩了很多的坑，因此这是一个比较重要的模块，我单独写一篇文章来记录。
- # 简要分析
- ## Docker的几种网络模式  
  1. **Bridge 模式**：
	- Bridge 模式是 Docker 默认的网络模式。在这种模式下，Docker 容器可以通过 Docker 守护进程的网络进行通信。每个容器都有自己的 IP 地址，并且可以通过容器名进行互相访问。
	  2. **Host 模式**：
	- Host 模式将容器连接到宿主机的网络堆栈，使得容器和宿主机共享同一个网络命名空间。在这种模式下，容器可以访问宿主机上的所有网络接口，且无需进行端口映射。
	  3. **None 模式**：
	- None 模式将容器连接到一个空的网络堆栈，即容器没有网络接口。这意味着容器内部无法进行网络通信，除非你手动添加网络接口或连接到其他网络。
	  4. **Overlay 模式**：
	- Overlay 模式用于跨多个 Docker 守护进程创建容器网络。这种模式通常用于容器集群中，以实现跨主机的容器通信。
	  5. **Macvlan 模式**：
	- Macvlan 模式允许你创建一个与宿主机相同的 MAC 地址的虚拟网卡，使得容器可以直接与宿主机网络中的其他设备进行通信，就像是物理设备一样。
	  6. **自定义网络模式**：
	- Docker 还允许用户创建自定义的网络模式，以满足特定的网络需求。可以通过 Docker 命令或 Docker Compose 文件来创建自定义网络模式。
	    
	  在我们使用过程中，比较常用的是`bridge`模式和`host`模式。`none`模式有时候很少使用，而我现在正在学习的`docker swarm`和`k8s`中则是使用到了`overlay`模式，我们创建的新容器如果不作规定，那么它便会默认为桥接模式，会给你一个它分配的网段作为这个网络的网段，在运行docker的服务器上如果使用`ifconfig`查看网络，则会发现会有很多`172.0.0.xxx`的网段，这个则是docker为其分配的网段。如下:  
	  ![image.png](https://wima.resoras.com/2024/03/20/65fa957e05e9d.webp)  
	  在使用过程中要尤为注意
- # 使用踩坑
- ## Nginx容器反代宿主机资源
- ### 问题阐述
  如果我们在宿主机上本地使用nginx,如果在现在我们运行了一个`whoami`服务，端口为`8080`,按照nginx的反代方式应该是  
  ```js
  server {
  listen         80 default_server;
  server_name    xxx.one;
  
  
  # 反向代理到 whoami
  location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_set_header X-Forwarded-For $remote_addr;
    client_max_body_size 128m;
  }
  }
  ```
  这样反代如果服务正常是可以访问的，但是如果是在Nginx容器内部这么配置就会出现问题，如果是在`Host`模式下，这么配置因为`Host`模式是和本机共用网络，则还是可以生效的，但是如果是在`bridge`模式下，因为容器内部是桥接的网段，不合宿主机本身共用网络，你使用的`localhost`是容器内部的本地，而不是宿主机的本地，于是就会无法反代。
- ### 解决办法
  我们知道，桥接模式下会给每个容器分配一个网段，在容器内部，假设分配的网段是`172.0.0.0/16`,那么在容器内部则`172.0.0.1`就会变成宿主机的`ip`,我们可以通过这个`ip`和宿主机沟通。以上配置则可以改为  
  ```js
  server {
  listen         80 default_server;
  server_name    xxx.one;
  
  
  # 反向代理到 whoami
  location / {
    proxy_pass http://172.0.0.1:8080;
    proxy_set_header X-Forwarded-For $remote_addr;
    client_max_body_size 128m;
  }
  }
  ```
  那么问题是如何知道这个容器网络的`ip`,我们当然可以创建之后通过一些命令来查看这个`ip`,但是这样太麻烦了，而且不好管理，因为在创建`network`时候分配的网段是随机的，如果你需要停止以下某个容器，然后再启动可能网段就变化了。所以我们可以提前指定，`Docker`是允许提前指定`ip`的，如下  
  ```shell
  docker network create --subnet=172.18.0.0/16 nginx_default
  docker run -d --name nginx_test --net nginx --ip 172.18.0.2 nginx
  ```
  上面这两条命令的操作是:
- 创建一个网络，名为`nginx_default`，并且设置网段为`172.18.0.0/16`
- 创建一个容器，绑定网络为上述网络，`ip`为`172.18.0.2`这样即可方便的在容器内部通过`172.18.0.1`进行沟通。
  
  compose的实现方式则为  
  ```yaml
  version: '3'
  
  networks:
  nginx_default:
    ipam:
      driver: default
      config:
        - subnet: 172.18.0.0/16
  
  services:
  nginx_test:
    image: nginx
    networks:
      nginx_default:
        ipv4_address: 172.18.0.2
  
  ```
- ## 容器间沟通
  上述是需要你反代宿主机的某个端口下的服务时候，会需要以上操作。但是如果这个服务也是一个容器，则有更好的办法。因为`docker`的桥接网络是一个的网段，我们一个网络可以添加很多个容器，一个容器也可以加入很多个网络。如果两个容器都加入了`nginx_default`网络，我们只需要通过该容器的名称来进行容器间沟通即可。如果使用nginx反代在同一个网络下端口为8080的`whoami`容器，即为以下操作:  
  ```shell
  docker network create nginx_default
  docker run -d --name web --network nginx_default web_image:latest
  docker run -d --name nginx \
           --network nginx_default \
           -v ./nginx.conf:/etc/nginx/nginx.conf:ro \
           -p 80:80 \
           nginx:latest
  ```
  然后修改`nginx`配置如下  
  ```js
  # vi ./nginx.conf
  server {
    listen 80;
    server_name localhost;
  
    location / {
        proxy_pass http://web:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
  }
  ```
  
  使用compose配置则如下，nginx配置不变:  
  ```yaml
  version: '3'
  
  services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - nginx_default
  
  web:
    image: web_image:latest
    container_name: web
    networks:
      - nginx_default
  
  networks:
  nginx_default:
    driver: bridge
  
  ```
  这种方式因为是在容器间沟通，你可以不将要反代的服务的端口暴漏出来，会增加一定的安全性。
- # `Docker compose`的网络配置方法
- ## 默认配置
  ```yaml
  version: '3'
  
  services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
  
  web:
    image: web_image:latest
    container_name: web
  ```
  这种配置方式会自动创建网络，在`stack`下的所有容器都是在同一个桥接网络，即nginx和web都在同一个网络
- ## 自定义配置
  ```yaml
  version: '3'
  
  services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - nginx_default
  
  web:
    image: web_image:latest
    container_name: web
    networks:
      - nginx_default
  
  networks:
  nginx_default:
    driver: bridge
  ```
  可以自定义网络的名称，属性以及一些配置。
- ## 外部网络
  ```yaml
  version: '3'
  
  services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    networks:
      - nginx_default
      - dashboard  # 将nginx容器连接到dashboard网络
  
  web:
    image: web_image:latest
    container_name: web
    networks:
      - nginx_default
  
  networks:
  nginx_default:
    driver: bridge
  dashboard:  # 定义一个名为dashboard的外部网络
    external: true  # 声明这是一个外部网络
  ```
  这个配置中多加了一个`dashboard`，我们可以添加`external: true`申明它是一个外部网络，这样它就不会自己创建，如果外部没有这个网络则需要提前创建，不然会报错。这个配置的应用则是需要一个容器加入多个网络的场景，比如:  
  现在有一个`dashboard`容器，使用的是其他`compose`文件或者使用的是命令，网络也不是`nginx_default`则，可以声明一个外部网络，然后加入这个外部网络使nginx容器也可以沟通这个容器，从而可以反代。这个外部网络是`dashboard`的网络，当然反过来也是可以的。  
  
  有人会问，为什么不全部都使用`nginx_default`呢，我认为主要有以下几点
- 不便管理，如果容器多了也会出现用满的情况
- 所有服务都暴漏，相互之间都可以沟通，有安全风险
- # 代码示例
  以上如果理解不透彻可以使用以下的编写的示例来稍微理解一下，这是一段`matrix synapse`的部署配置，会在后续出详细教程。
- ## `Traefik`配置
  ```yaml
  # 默认创建的network为traefik_default
  version: '3'
  
  services:
  traefik:
    image: traefik
    command: --api.insecure=true --providers.docker
    restart: always
    container_name: traefik
    ports:
      - "80:80"
      - "443:443"
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: '2048M'
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./config:/etc/traefik
      - ./letsencrypt:/letsencrypt:rw
  ```
- ## `Matrix`配置
  
  ```yaml
  version: "3.4"
  
  networks:
  matrix_network:
    external: true
  traefik_default:
    external: true
  
  services:
  nginx:
    image: "nginx:latest"
    restart: "unless-stopped"
    ...
    networks:
      - traefik_default
      - matrix_network
  
  ntfy:
    image: binwiederhier/ntfy
    ...
    restart: unless-stopped
    networks:
      - traefik_default
  
  stickerpicker:
    image: code.loveblancs.com/sora/stickerpicker:latest
    restart: always
    container_name: sticker      
    ...
    networks:
      - traefik_default
  
  pushnits:
    image: ghcr.io/pushbits/server:latest
    ...
    networks:
      - traefik_default
      - matrix_network
  
  synapse:
    hostname: synapse
    container_name: synapse
    image: matrixdotorg/synapse:latest
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    environment:
      TZ: Asia/Shanghai
      ...
    networks:
      - traefik_default
      - matrix_network
  
  mautrix-telegram:
    container_name: mautrix-telegram
    image: dock.mau.dev/mautrix/telegram
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./Matrix/mautrix-telegram:/data
    networks:
      - matrix_network
      - traefik_default
  
  matrix-gmessage:
    container_name: matrix-gmessage
    image: dock.mau.dev/mautrix/gmessages
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./Matrix/matrix-gmessage:/data
    networks:
      - matrix_network
      - traefik_default
  
  matrix-wechat:
    hostname: matrix-wechat
    container_name: matrix-wechat
    image: lxduo/matrix-wechat:2
    restart: unless-stopped
    ports:
      - "20002:20002"
    depends_on:
      synapse:
        condition: service_healthy
    volumes:
      - ./Matrix/matrix-wechat:/data
    networks:
      - matrix_network
      - traefik_default
  
  mautrix-discord:
    container_name: mautrix-discord
    image: dock.mau.dev/mautrix/discord
    restart: unless-stopped
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - ./Matrix/mautrix-discord:/data
    networks:
      - matrix_network
      - traefik_default
  
  postgres:
    hostname: postgres
    container_name: postgres
    image: postgres:14
    restart: always
    volumes:
      - ./Matrix/postgres/create_db.sh:/docker-entrypoint-initdb.d/20-create_db.sh
      - ./Matrix/postgres/data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U synapse_user"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - matrix_network
  ```
- ## `Nginx`配置
  ```js
  server {
  listen         80 default_server;
  server_name    xxx.com;
  
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
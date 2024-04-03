- [[Feb 8th, 2024]] #代理
- # 前言
  目前市面上可使用的反向代理工具主要有Caddy, [[Nginx]] ,Traefik等，在我学习历程中首先使用了Nginx作为反向代理工具，Nginx的优点是性能好，缺点则是配置麻烦，并且不可以很方便的配置证书，于是在后来使用了Caddy进行反代，虽然性能一言难尽，但是可以自己获取证书以及配置简单。在接触到云原生以后转为使用Traefik进行动态的配置，优点是配置方便，与Docker这种容器强绑定，缺点就是有些复杂情况还是要使用Nginx作为下层代理(当然也有可能是我太菜)。具体联动可以查看 [[Traefik和Nginx及其双重代理]]
- # 介绍
  Traefik 是一个 HTTP 反向代理和负载均衡器，可促进微服务的部署。它可以与现有的基础设施组件（如 Docker、Swarm Mode、Kubernetes、Consul、Etcd、Rancher v2、Amazon ECS）无缝集成，并自动动态地进行自我配置。Traefik 在云、本地和混合云中用作路由、负载均衡和代理的现代标准。反向代理和正向代理的区别可以查看[[反向代理和正向代理]]
- # 单层Traefik
  
  ```yaml
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
      - /var/run/docker.sock:/var/run/docker.sock #挂载Docker套接字
      - ./config:/etc/traefik #挂载Traefik配置
      - ./letsencrypt:/letsencrypt:rw # 证书存储的地址
  ```
  
  使用Docker compose方式配置，这里创建了一个名叫Traefik的容器。在这里要注意config文件夹权限的问题，可能会出现无法读取，如果嫌麻烦直接无脑`sudo chmod 777 -R config`即可。同时要在`config/traefik.yml`文件中写入以下配置  
  
  ```yaml
  api:
  # 开启 WEB UI
  dashboard: false
  # 安全模式
  insecure: true
  
  
  # 发现docker或者file文件中定义的服务
  providers:
  # 监听file
  file:
    # 定义动态配置文件所在的文件目录(容器内部路径)
    directory: /etc/traefik/config
    # 监听动态配置文件的变更
    watch: true
  
  
  # 监听docker
  docker:
    # 如果置为false，那么docker容器需要在labels中声明traefik.enable=true，否则容器会被忽略
    exposedByDefault: false
  
  
  # 定义流量入口(也就是对外暴露的监听的端口，该处定义的端口需要在docker-compose.yml中做端口暴露映射)
  entryPoints:
  # 定义一个名称为http的入口，监听80端口，由80端口进入的流量都由它来代理
  http:
    address: ":80"
  
  # 定义https入口，监听443端口，由80端口进入的流量都由它来代理
  https:
    address: ":443"
  
  # mysql的tcp代理入口
  mysql:
    address: ":3306"
  
  # redis的tcp代理入口
  redis:
    address: ":6379"
  
  # 开启ACME 自动生成HTTPS证书
  certificatesResolvers:
  myCertResolver:
    acme:
      # 邮箱地址
      email: "ssl@exa.com"
      # 签发的https证书存放位置
      storage: "/letsencrypt/acme.json"
      # 自动签发证书的一种验证方式(还有tlsChallent、dnsChallenge,我们用的是常用的这种httpChallenge方式)
      httpChallenge:
        entryPoint: http
  
  middlewares:
  hsts:
    headers:
      stsSeconds: 15552000
      stsIncludeSubdomains: true
  
  
  myheaders:
    headers:
       customResponseHeaders:
         Allow: ".*"
       accessControlAllowHeaders: ".*"
  ```
  
  这里定义了四个入口点分别是http,https,mysql,redis,在你反代时候可以通过这几个入口点进行反代。下方的则是ssl证书的申请，邮箱乱写无所谓，只是用来接受证书通知的。配置好以上之后，如果想要反代一个容器，则只需要在其启动时使用label语句申明,如下  
  
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
    volumes:
      - "./nginx/matrix.conf:/etc/nginx/conf.d/matrix.conf"
      - ./nginx/www:/var/www/
  # 申明语句
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.matrix-nginx.rule=Host(`xxx.one`)"
      - "traefik.http.routers.matrix-nginx.service=matrix-nginx"
      - "traefik.http.routers.matrix-nginx.entrypoints=https"
      - "traefik.http.services.matrix-nginx.loadbalancer.server.port=80"
      - "traefik.http.routers.matrix-nginx.tls=true"
      - "traefik.http.routers.matrix-nginx.tls.certresolver=myCertResolver"
      - "traefik.http.services.matrix-nginx.loadbalancer.passhostheader=true"
      - "traefik.docker.network=traefik_default"
    networks:
      - traefik_default
      - matrix_network
  ```
  
  这样Traefik就会自动发现这个服务并反代，我这里编写了一个很完整的申明，在自己使用过程中可以选择合适的进行省略。
-
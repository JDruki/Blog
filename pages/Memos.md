- [[Mar 27th, 2024]] #Docker #生产力
- # 介绍
  Memos 是一项隐私优先的轻量级笔记服务。轻松捕捉和分享您的好主意。
- # 部署
  官网使用的是 [[Docker]] 部署的，但是我们为了方便使用 [[docker compose]]一键安装，参考以下配置。  
  ```yaml
    memos:
      image: neosmemo/memos:stable
      container_name: memos
      volumes:
        - ./Memos/memos/:/var/opt/memos
  ```
  但是官网使用的是sqlite，如果有需求则需要切换为MYSQL，具体参照[[Memos切换MYSQL数据库]],加入环境变量  
  ```yaml
      environment:
        - MEMOS_DRIVER=mysql
        - MEMOS_DSN=root:mY27fd6r2jfdLnG8B_@tcp(memosdb)/memosdb
  ```
  我一般使用Traefik,参考 [[Traefik]] 中的配置，如果配置好则只需要在compose加入label语句，如下  
  ```yaml
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.memos.rule=Host(`snote.xxx.com`)"
        - "traefik.http.routers.memos.entrypoints=https"
        - "traefik.http.routers.memos.service=memos"
        - "traefik.http.routers.memos.tls=true"
        - "traefik.http.services.memos.loadbalancer.server.port=5230"
        - "traefik.http.routers.memos.tls.certresolver=myCertResolver"
        - "traefik.http.services.memos.loadbalancer.passhostheader=true"
  ```
  完整配置  
  ```yaml
    memos:
      image: neosmemo/memos:stable
      container_name: memos
      volumes:
        - ./Memos/memos/:/var/opt/memos
      environment:
        - MEMOS_DRIVER=mysql
        - MEMOS_DSN=root:password@tcp(memosdb)/memosdb
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.memos.rule=Host(`snote.xxx.com`)"
        - "traefik.http.routers.memos.entrypoints=https"
        - "traefik.http.routers.memos.service=memos"
        - "traefik.http.routers.memos.tls=true"
        - "traefik.http.services.memos.loadbalancer.server.port=5230"
        - "traefik.http.routers.memos.tls.certresolver=myCertResolver"
        - "traefik.http.services.memos.loadbalancer.passhostheader=true"
  ```
- 具体使用可以参考[[Memos使用技巧]]
-
- #Docker #折腾日记 #Self-hosted
-
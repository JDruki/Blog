# 介绍
Uptime Kuma是一个开源的网络服务监控工具。它允许用户监视他们的网络服务，以确保其正常运行，并提供有关服务可用性和性能的实时信息。Uptime Kuma提供直观的用户界面，支持多种通知方式，可以通过配置来满足用户对监控的需求。
- # 部署
  Uptime Kuma可以很方便的使用 [[docker compose]] 来进行部署
- ## 配置文件
  ```shell
  mkdir UptimeKuma&&cd UptimeKuma
  vi compose
  ```
  在其中输入  
  ```yaml
  version: '3'
  services:
    uptimekuma:
      image: louislam/uptime-kuma
      container_name: uptime-kuma
      ports:
        - 3001:3001
      volumes:
        - ./Uptime_Kuma:/app/data
        - /var/run/docker.sock:/var/run/docker.sock
      restart: always
  ```
- ## 启动
  ```shell
  docker compose up -d
  ```
- ## 查看日志
  ```shell
  docker compose logs -f
  ```
- # 使用方法
  参考[官网](https://uptime.kuma.pet)
- # 联动
	- 可以和[[PushBits]]实现使用PushBits来将通知转发到 [[Matrix]]
	- 可以和 [[Homeassistain]]联动
	-
- #Docker #[[docker compose]] #折腾日记 #Self-hosted
-
-
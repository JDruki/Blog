- #Docker #博客
- # 前言
  之前我们已经安装了 [[Docker]] 和 [[docker compose]] ,并对其进行了一些最基础的配置，这里便记录一些Docker最常用的命令
-
- # Docker 的基本使用方法：
- ## 常用命令汇总
- ### 启动容器
  使用以下命令可以启动一个容器，其中，`OPTIONS` 是一些启动容器时的选项，`IMAGE` 是指定要使用的镜像，`COMMAND` 和 `ARG` 是可选的命令和参数。  
  ```shell
  
  docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
  ```
- #### 绑定挂载方式
  ```shell
  docker run -d -p 80:80 --name mynginx -v /root/nginx:/etc/nginx --network nginx_default nginx
  ```
  这个命令的解释如下：
- `-d`：表示在后台运行容器。
- `-p 80:80`：将主机的80端口映射到容器的80端口，这样就可以通过主机的80端口访问Nginx服务。
- `--name mynginx`：为容器指定一个名称，这里命名为 `mynginx`。
- `-v /root/nginx:/etc/nginx`：将主机上的 `/root/nginx` 目录挂载到容器内的 `/etc/nginx` 目录，这样可以在主机上编辑Nginx配置文件并立即生效。
- `--network nginx_default`指的是容器的网络为nginx_default,在你需要多个容器相互沟通时候需要指定此项，注意需要提前运行`docker network create nginx_default`创建网络。
- `nginx`：指定要使用的Nginx镜像。
- #### Volume方式
  ```shell
  docker volume create nginx-config
  docker run -d -p 80:80 --name mynginx -v nginx-config:/etc/nginx nginx
  ```
  其他就不多做解释，第一条命令主要是提前创建一个volume,第二条命令启动容器并且将volume申明给mynginx容器。
- ### 查看/停止正在运行的容器
  使用`docker ps`查看正在运行的服务，如下  
  ![image.png](https://wima.resoras.com/2024/03/20/65fa8bf30e9ad.webp)  
  在这里我们只需要注意第一行这个ContainerID即可，其他的可以自己去查含义。  
  
  如果我们需要删除这个容器只需要输入`docker stop containerid`即可，注意这时只是停止了这个容器，如果需要删除还需要使用`docker rm containerid`,如下  
  ![image.png](https://wima.resoras.com/2024/03/20/65fa8cc4dc189.webp)
- ### 进入容器内部执行命令
  有时候我们需要在容器内部执行一些命名用来调试，这时候可以使用`docker exec -it containerid bash`或者`docker exec -it containerid sh`进入容器内部  
  ![image.png](https://wima.resoras.com/2024/03/20/65fa8d5733b8e.webp)
- ### 查看容器日志
  想要查看容器日志只需要运行`docker logs container`即可，你也可以在命令之后加上`-f`来实现实时输出日志。  
  ![image.png](https://wima.resoras.com/2024/03/20/65fa8dda15505.webp)
- ### 其他命令
  这些命令一般人使用比较少，可以自己了解  
  ```shell
  docker network ls #查看所有网络
  docker network rm/crtate {network_name} #删除/创建某个网络
  docker volume ls 
  docker volume rm/create {volume_name} #同上
  docker pull {容器名}:{tag} #拉取某个容器，一般在使用run命令时会自动执行
  docker push {} #上传某个镜像
  docker build -it {image_name}:{tag} . #构建镜像
  ```
- # Docker compose基本使用方法
- ## 简单介绍
  Docker compose比Docker的优点是他是通过在一个`yaml`文件来创建镜像的，可以很方便的创建和维护一个盏，使用方法只需要随便找一个目录，然后使用`vi`编辑器创建一个`compose.yml`文件,然后使用docker compose up -d命令启动即可，不加`-d`容器只会前台运行。加了之后容器会后太运行，例如  
  ```shell
  vi compose.yml #创建文件
  docker compose up -d #启动
  docker compose logs -f #查看日志
  ```
- ## 实例讲解
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
      - /var/run/docker.sock:/var/run/docker.sock
      - ./Traefik/config:/etc/traefik
      - ./Traefik/letsencrypt:/letsencrypt
  ```
  
  这个文件定义了一个名为 `traefik` 的服务，以下逐个分析：  
  
  1. `version: '3'`：
	- 这表示使用的是 Docker Compose 文件的版本号。当前版本是3。
	    
	  2. `services:`：
	- 定义了一个或多个服务，每个服务都是一个独立的容器。
	    
	  3. `traefik:`：
	- 定义了一个名为 `traefik` 的服务。
	    
	  4. `image: traefik`：
	- 指定了要使用的镜像，这里是 Traefik。
	    
	  5. `command: --api.insecure=true --providers.docker`：
	- 定义了容器启动时要执行的命令。在这里，Traefik 将以不安全模式启动，并启用 Docker 作为提供者。
	    
	  6. `restart: always`：
	- 定义了容器退出时的重新启动策略，这里是总是重新启动。
	    
	  7. `container_name: traefik`：
	- 指定了容器的名称。
	    
	  8. `ports:`：
	- 定义了容器内部端口与宿主机的端口映射关系。
	- `"80:80"` 将容器的80端口映射到宿主机的80端口。
	- `"443:443"` 将容器的443端口映射到宿主机的443端口。
	    
	  9. `deploy:`：
	- 这个部分定义了部署配置，包括资源限制等。
	    
	  10. `resources:`：
		- 定义了容器的资源限制。
		- `limits:` 指定了资源的上限。
		- `cpus: '0.5'` 指定了CPU的限制为0.5核。
		- `memory: '2048M'` 指定了内存的限制为2048MB。
		    
		  11. `volumes:`：
		- 定义了容器内部的挂载卷。
		- `/var/run/docker.sock:/var/run/docker.sock` 将宿主机的 Docker 守护进程的 Unix 套接字挂载到容器内，以便 Traefik 可以与 Docker 守护进程交互。
		- `./Traefik/config:/etc/traefik` 将宿主机上的 `./Traefik/config` 目录挂载到容器内的 `/etc/traefik` 目录，用于存放 Traefik 的配置文件。
		- `./Traefik/letsencrypt:/letsencrypt` 将宿主机上的 `./Traefik/letsencrypt` 目录挂载到容器内的 `/letsencrypt` 目录，用于存放 Let's Encrypt SSL 证书。
		    
		  启动服务：  
		  ```bash
		  docker-compose up -d
		  ```
		    
		  停止服务：  
		  ```bash
		  docker-compose down
		  ```
		    
		  查看日志：  
		  ```bash
		  docker-compose logs -f
		  ```
		    
		  进入容器：  
		  ```bash
		  docker exec -it traefik sh
		  ```
		    
		  在进入容器后，你可以执行各种命令来查看和管理 Traefik 服务。
-
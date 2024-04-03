- [[Feb 7th, 2024]] #Docker #博客
- # 前言 
  在容器化应用程序时，通常会遇到需要持久存储数据的情况，比如数据库文件、应用程序日志、配置文件等，我们通常会使用Voklume或者绑定挂载的方式进行持久化数据，当然在使用K8s或者Docker Swarm过程中还会使用到一种方式便是使用远程存储的方式，在个人使用过程不做考虑。
- # **数据卷 (Volumes)**:
  
  Volumes 允许容器将特定路径的数据存储在主机上的目录中，这些数据将在容器重新创建或删除时得以保留。Volumes 提供了一种简单而有效的方法来管理容器中的数据，并且与容器的生命周期分离，使得容器可以更轻松地迁移、复制或删除而不会丢失重要数据。数据卷可以手动创建，也可以在容器启动时自动创建。数据卷可以用于持久存储数据，同时也可以用于容器之间共享数据
- ## 示例:
  
  创建volume  
  
  ```shell
  docker volume create config
  ```
  
  使用Docker compose管理容器  
  
  ```yaml
  version: '3'
  # 申明一个volume
  volumes: 
  config:
  
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
      - config:/etc/traefik # 映射
  ```
  在这里我们创建了一个config的volume,同时将容器内部的文件绑定到这个volume，当你停止这个容器时候，volume中的数据不会丢失。同时多个容器可以共享同一个 Volume，使得容器之间可以共享数据。但是由于volume是通过docker管理的，保存在宿主机，如果跨服务器之间进行备份恢复的话会比较麻烦于是我们使用大多以下方式进行容器数据持久化。
- # **绑定挂载 (Bind Mounts)**:
  绑定挂载是将主机上的目录或文件直接挂载到容器中的特定路径上。与数据卷不同，绑定挂载直接映射到主机文件系统上的目录，因此对于主机文件系统的更改会直接影响到容器内的数据。绑定挂载适用于需要与主机文件系统共享数据的场景。
- ## 示例
  这里同样使用docker compose方式  
  
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
      - ./config:/etc/traefik
  ```
  在这里我们将traefik容器中的/etc/traefik目录映射到宿主机当前目录下的config目录，和volume方式相同，多个服务可以映射同一个文件夹，但是在映射途中要注意权限的问题，有时候会出现无法读取的情况。这种方式下我们就可以很便利的将这个目录备份之后在各个服务器之间迁移。
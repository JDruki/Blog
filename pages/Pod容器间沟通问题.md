- #Podman #运维 #问题记录
- 经过之前的[[深入理解容器间沟通]]，我们知道容器间沟通通过的是dns解析的方式，使用以下命令创建容器发现DNS解析出错，原本是容器的ip被解析成为了宿主机的ip,原因还在 [[TODO]]
	- ```shell
	  
	  podman run -d \
	    --name=librum_db \
	    --network=librum \
	    --pod new:librum\
	    --replace \
	    -v ./librum_db:/var/lib/mysql:z \
	    --restart=unless-stopped \
	    mariadb:latest
	  ```
	- --pod new:librum\
- 目前解决办法，先创建`pod`再创建容器，如下
  ```shell
  podman pod create librum
  podman run -d \
    --name=librum_db \
    --network=librum \
    --pod librum\
    --replace \
    -v ./librum_db:/var/lib/mysql:z \
    --restart=unless-stopped \
    mariadb:latest
  ```
- # 更新
  在查阅资料之后找到了原因，有以下原因
	- 同一 pod 中的所有容器共享 IP 地址、MAC 地址和端口映射。
	- 使用 localhost:port 表示法在同一 pod 中的容器之间进行通信。
	- ## 理解
	  也就是说在同一个Pod下面所有的容器是共享网络的，与以前的`Docker compose`方式使用有很大出入，所以如果需要沟通，则直接使用`localhost`进行沟通即可，如果需要`Pod`间沟通，则需要将`Pod`关联到一个网络中，由此便明白了。
	- 参考[Pod间沟通](https://access.redhat.com/documentation/zh-cn/red_hat_enterprise_linux/9/html/building_running_and_managing_containers/proc_communicating-between-two-containers-in-a-pod_assembly_communicating-among-containers)
	- 注意Pod间沟通时候pod的名字不要和容器的名字相同，会出现问题，切记切记#card
	- [[Mar 29th, 2024]]
	-
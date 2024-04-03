- #Kubernetes #运维 #TODO
- # 前言
- # 准备
	- 硬件准备
	  准备至少2台以上服务器，一台`manage`一台`node`，我这里准备了五台服务器，按照以下排列，其实node可以更多，只是囊中羞涩。
	- {{renderer excalidraw, excalidraw-2024-04-03-17-08-21}}
	- ## 服务器环境安装
		- ### 拉取配置
			- 首先安装一些必要的环境，我这里安装了`git`和`python3.12`
			  ```shell
			  dnf install git python3.12 python3.12-pip
			  ```
			- 然后克隆`kubespray`的官方仓库
			  ```shell
			  git clone --depth=1 https://github.com/kubernetes-sigs/kubespray.git&&cd kubespray
			  ```
			- 安装依赖，因为我的`python`是3.12的所以不使用默认的命令
			  ```shell
			  sudo pip3.12 install -r requirements.txt
			  ```
		- ### 修改配置
			- 在当前目录下，复制一份配置文件，文件名可另取
			  ```shell
			  cp -rfp inventory/sample inventory/soras
			  ```
			- 使用 [[Vim]]修改配置
			-
- # 启动集群
- # 参考文献
  [多种方法部署Pandora，让ChatGPT更好用 - 梅塔沃克 - 专注跨境](https://omnivore.app/me/https-iweec-com-882-html-188852229b8)  
  内容
- [自建Kubernetes集群完全指南（含云主机公网部署方案）](https://omnivore.app/me/https-www-karsonjo-com-deploy-a-self-hosted-kubernetes-18ea32a7d0a)
-
- [[Apr 3rd, 2024]]
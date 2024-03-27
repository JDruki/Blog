- #名词解释
- # 介绍 
  Compose 是用于定义和运行多容器 Docker 应用程序的工具。通过 Compose，您可以使用 YML 文件来配置应用程序需要的所有服务。然后，使用一个命令，就可以从 YML 文件配置中创建并启动所有服务。目前compose已被集成在 [[Docker]]官方，所以无需单独安装compose。
- # 使用步骤
	- 使用 Dockerfile 定义应用程序的环境。
	- 使用 compose.yml 定义构成应用程序的服务，这样它们可以在隔离环境中一起运行。
	- 最后，执行 docker-compose up 命令来启动并运行整个应用程序。
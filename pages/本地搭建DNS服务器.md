- #Linux #运维
- ```shell
  sudo podman run --name adguardhome \
      --restart unless-stopped \
      -v ./workdir:/opt/adguardhome/work:z \
      -v ./confdir:/opt/adguardhome/conf:z \
      -p 53:53/tcp -p 53:53/udp \
      -p 3000:3000/tcp \
      -d adguard/adguardhome
  ```
- [[Apr 7th, 2024]]
-
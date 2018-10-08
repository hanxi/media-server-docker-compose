# media-server-docker-compose

多媒体服务器搭建

使用 docker-compose 组合各个 docker

- Aria2 + AriaNg +  FileBrowser 来自 [wahyd4/aria2-ariang-docker](https://github.com/wahyd4/aria2-ariang-docker)
- Aria2 bt tracker 来自 [hanxi/aria2-bt-tracker](https://github.com/hanxi/aria2-bt-tracker)
- Minidlna 来自 [vladgh/docker_base_images/tree/master/minidlna](https://github.com/vladgh/docker_base_images/tree/master/minidlna)
- Samba 来自 [jcbiellikltd/docker-samba-server](https://github.com/jcbiellikltd/docker-samba-server)

## 启动

```
git clone https://github.com/hanxi/media-server-docker-compose.git
cd media-server-docker-compose
docker-compose up -d
```

## 关闭

```
docker-compose stop
```

## 各个 docker 配置介绍和使用

所有容器都以主机的 `/data/` 目录为资源存放的文件夹，如果主机是 Linux， /data 文件夹可自行 mount 为某个磁盘，如果主机使用的是 boot2docker, 可以在 virtual box 里面配置共享目录并自动挂载

### aria2

```
    container_name: aria2   # 容器名字
    image: wahyd4/aria2-ui  # 镜像名字
    ports:                  # 端口映射，主机:容器
      - "8000:80"
      - "443:443"
      - "6800:6800"
    volumes:                # 磁盘映射，主机:容器
      - /data:/data:rw
    environment:            # 环境变量
      - DOMAIN=:80
      # - SSL=true
      - RPC_SECRET=HelloWorld   # jsonrpc 的 token
      - ARIA2_USER=admin        # aria2 的用户名
      - ARIA2_PWD=12345678      # aria2 的登录密码
      - ENABLE_AUTH=true        # 开启 aria2 密码验证
    restart: always
```

- 访问 http://hostip:8000 进入 AriaNg
 - 用户名为： admin
 - 密码为：12345678
 - 进入 AriaNg 设置，修改 Aria2 RPC 密钥为 HelloWorld
- 访问 http://hostip:8000/files 进入 FireBrowser
 - 用户名和密码为： admin
 - 进入设置，修改语言为中文，并修改密码
    
### bt-tracker

```
    container_name: bt-tracker
    image: hanxi/aria2-bt-tracker
    environment:
      - ARIA2_URL=http://aria2:6800/jsonrpc
      - ARIA2_TOKEN=HelloWorld # 对应上面 aria2 配置的 RPC_SECRET
      - TRACKER_URL=https://raw.githubusercontent.com/ngosang/trackerslist/master/trackers_all.txt
    links:
      - aria2:aria2 # 提供直接访问 aria2 容器，对应上面配置的 ARIA2_URL
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
    restart: always
```

- 默认每天凌晨4点更新 bt-tracker，如果想要修改，可以自己编译镜像。后面可能会把这个提成环境变量来配置。
- `TRACKER_URL` 可以配置成这里的 [ngosang/trackerslist](https://github.com/ngosang/trackerslist) 列出的
    
### minidlna

```
    image: vladgh/minidlna
    container_name: minidlna
    network_mode: 'host'  # 网络模式设为 host
    environment:
      - MINIDLNA_MEDIA_DIR=/media
      - MINIDLNA_FRIENDLY_NAME=MiniDLNA
      - MINIDLNA_INOTIFY=yes
      - MINIDLNA_NOTIFY_INTERVAL=3
      - MINIDLNA_NETWORK_INTERFACE=eth1
    volumes:
      - /cache/minidlna:/var/lib/minidlna
      - /data:/media
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
    restart: always
```

- 如果 docker 是运行在 boot2docker（windows/macosx） 里的，则网络模式采用 host，用端口映射会有问题
- `MINIDLNA_NETWORK_INTERFACE=eth1` 这个需要根据自己的虚拟机网卡情况来修改，这里有介绍 [#note-for-macos](https://github.com/viranch/docker-minidlna/blob/master/README.md#note-for-macos)

> If you're using docker on MacOS (this usually means having a VirtualBox VM with a docker installation, which is done by the docker MacOS installer), you need to do some extra steps to get this working. DLNA server & clients need to be on same network but the VirtualBox VM has a NATed network adapter. You need to add a new bridged network adapter:
> 1. Stop the VM: `docker-machine stop default`
> 2. Open VirtualBox, select the VM with name "default" and go to its Settings -> Network -> Select an unused Adapter, enable it and select "Bridged Adapter" under "Attached to:" field.
> 3. Start the VM: `docker-machine start default`
> 4. From VirtualBox, again select the "default" VM and click on Show. This will open up the terminal of the VM, use `ifconfig` to figure out the new bridged network adapter, say it is `eth2`.
> 5. Now add `-e NIC=eth2` in the above run command after the `-v` switch.

### samba

```
    container_name: samba
    image: joebiellik/samba-server
    network_mode: 'host'  # 网络模式设为 host
    volumes:
      - /data:/mnt
      - ./smb.conf:/etc/samba/smb.conf
    environment:
      - USERNAME=samba # 用户名
      - PASSWORD=samba # 密码
    logging:
      driver: "json-file"
      options:
        max-size: "1m"
    restart: always
```

- 主要是网络问题需要跟 minidlna 一样处理

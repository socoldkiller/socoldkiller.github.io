---
title: "内网穿透实战：从 frp 到 tailscale 的完整部署指南"
date: 2025-08-16
draft: false
tags: ["网络", "内网穿透", "frp", "tailscale", "headscale", "Docker", "VPN", "网络代理"]
categories: ["网络工具", "系统运维"]
summary: "深入对比 frp、tailscale、nps、zeroTier、cloudflare zero-trust 等主流内网穿透方案的优劣，并详细介绍如何使用 Docker 部署 frp 和 tailscale+headscale 解决方案，包含完整配置步骤和网络拓扑说明。"
---

## 组网方案概览
**优缺点总结**
| 方案                    | 优点                                                                     | 缺点                                              | 应用范围  | 地址                                            |
| --------------------- | ---------------------------------------------------------------------- | ----------------------------------------------- | ----- | --------------------------------------------- |
| frp                   | 部署简单，需要权限少，学习成本低                                                       | 功能单一，如果需要发布局域网多个IP，则需要一个一个配置，这样比较繁琐             | 家庭，公司 | https://github.com/fatedier/frp.git           |
| tailscale             | 配置完成即可享受上级路由器为公网IP的待遇，后续基本0配置，p2p能力一流，穿透成功即不需要公网中转流量，生态好，配合k3s可以做到多云组网 | 部署比较复杂，权限多，学习成本高                                | 公司    | https://github.com/socoldkiller/tailscale.git |
| nps                   | 部署简单，需要权限少，学习成本低                                                       | 同frp，（不过好像有界面）                                  | 家庭    | https://github.com/ehang-io/nps.git           |
| zeroTier              | 基本同tailscale                                                           | 自建的moon服务器不好用，用官方服务器打洞失败，要忍受 国内->国外服务器->国内的超长延迟 | 家庭，公司 | https://www.zerotier.com/                     |
| cloudflare zero-trust | 部署超级简单，家庭组网首选，基于zero-trust（零信任安全），穿透需要认证，比frp更加安全,                     | 同frp                                            | 家庭    | https://one.dash.cloudflare.com/              |


上述所有的方案均提供断线重连，配合systemd | docker daemon 能做到局域网突然没网然后有网也能自动重连。其中tailscale,zerotier 因为是商业化出品，且都是国外产品，所以中心服务器都在国外，连接比较困难，所以需要自建中心服务器。


总而言之:
frp,nps                   是基于端口映射总的内网穿透
tailscale, zeroTier  是基于vpn做的内网穿透


下面着重介绍 frp,tailsacle 这两款工具的使用



**注：**
下面所述都是基于docker容器来构建的

## frp

### 服务端(frps)


1. 拉取 **frps**
```
docker pull  snowdreamtech/frps
```

2.  配置frps.toml 信息
    这里直接给出 **frps.toml** 文件供参考      
```
bindPort = 7000                                                                
quicBindPort = 7000                                                            
webServer.addr = "0.0.0.0"                                                     
```
 1. frps.toml是 frps的配置信息，具体可以看教程： https://gofrp.org 



4.  启动frps
     这里直接给出 **docker-compose.yml** 文件供参考
```
version: '3'
services:
  frps:
    image: snowdreamtech/frps
    container_name: frps
    network_mode: host
    restart: always
    volumes:
      - ./frps.toml:/etc/frp/frps.toml
```
  **注意**： 这里网络模式设置成host，让宿主机和docker 网络流量走同一张网卡

 1. 配置端口映射可以是可以，但某次添加端口的时候都要重新增加一次配置，还要重启docker 容器很麻烦，所以这边建议直接使用host模式，让容器走宿主机网络
 

这样服务端就配置完成了。


### 客户端(frpc)

1. 拉取frpc
```
docker pull snowdreamtech/frpc:latest
```


2. 配置frpc 信息，给出参考frpc.toml
	
```
serverAddr = "server.domain.com"                                               
serverPort = 7000                                                              
transport.protocol = "quic"                                                    
[[proxies]]                                                                    
name = "web-tcp"                                                               
type = "tcp"                                                                   
localIP = "127.0.0.1"                                                          
localPort = 443                                                                
remotePort = 443                                                              
```
 1. frpc.toml是 frps的配置信息，具体可以看教程： https://gofrp.org 
    上述配置就是把本地127.0.0.1 443端口发布到服务器443端口上


3. 启动frpc
       这里直接给出 **docker-compose.yml** 文件供参考
```
version: '3'                                                                   
services:                                                                      
  frpc:                                                                        
    image: snowdreamtech/frpc                                         
    container_name: frpc                                                       
    network_mode: host                                                         
    restart: always                                                            
    volumes:                                                                   
      - ./conf/frpc.toml:/etc/frp/frpc.toml
```

**注**:这里网络模式一定要设置成host，让宿主机和docker 网络流量走同一张网卡，（你也不想发布frpc里面的端口吧）

同样 如果非要是端口形式发布的，frpc.toml的localhost就要写你宿主机IP，通常是192.168.0.0/16 这个网段的，一定不要写127.0.0.1,这样就发布到frpc 内部容器的网络了



这样总的 frp就搭建完成了，即使这么简单的frp,配合反向代理也能完成很多花样来，后面在介绍



## tailscale 组网方案
 首先搭建headscale节点，同样使用docker 来部署

### headscale 

1. 拉取 headscale 镜像

```
docker pull ghcr.io/socoldkiller/headscale:dev
```

2.  运行容器
```
services:  
  headscale:  
    container_name: headscale  
    image: ghcr.io/socoldkiller/headscale:dev  
    restart: always  
    network_mode: host  
    volumes:  
      - ./container-config:/etc/headscale  
      - ./container-data:/var/lib/headscale  
      - ./run/headscale:/var/run/headscale  
    entrypoint: headscale serve
```

其中 **container-config** 文件夹下的config.yaml 至关重要.
以下代码是将config.yaml下载到本地
```
curl -o config.yaml https://raw.githubusercontent.com/juanfont/headscale/main/config-example.yaml
```

config.json 文档在 https://github.com/juanfont/headscale/blob/main/config-example.yaml 里

这里需要配置：
1. **server_url** 为你的反代网关的域名
2. **derp** 的server 开启
   其他大致不用变，按需求配置ACL

ACL的配置给出个最精简的版本
```
{
  "acls": [
    {
      "action": "accept",
      "src": [
        "*"
      ],
      "dst": [
        "*:*"
      ]
    }
  ],
  "autoApprovers": {
    "routes": {
      "10.42.0.0/16": [
        "root",
        "k3s"
      ],
      "192.168.0.0/16": [
        "root",
        "k3s"
      ],
      "10.0.0.0/8": [
        "root",
        "k3s"
      ]
    }
  }
}

```
上面表示接受所有的路由，并且字段批准 **10.42.0.0/16**和**192.168.0.0/16** 网段，用户名为**root**,**k3s**的用户



至此 服务端tailscale的中央处理器 headscale安装完成

~~什么，你问我UI? 那我只能说 如果你使用我的镜像，则web-ui已经直接集成在headscale上了，开箱即用。~~

### tailscale

相较于headscale，tailscale 安装很简单


```
services:
  tailscale:
    container_name: tailscale
    image: ghcr.io/tailscale/tailscale
    restart: always
    privileged: true
    devices:
      - /dev/net/tun
    network_mode: host
    volumes:
      - './tailscale/data:/var/lib' # 改成你的 tailscale 路径
      - '/dev/net/tun:/dev/net/tun'
    command: sh -c "mkdir -p /var/run/tailscale && ln -s /tmp/tailscaled.sock /var/run/tailscale/tailscaled.sock && tailscaled"
```
然后登录**headscale**
```
  docker exec tailscale tailscale up  --login-server=https://example.domain.ltd
```

其中有几个参数比较有用

1.  --accept-routes 通告当前路由，使得别的机器也能用过ip能访问到你的机器
2. --advertise-routes 通告网段路由，你的机器作为路由器，别的机器访问你网段的服务器，流量从你这里中转
3. --advertise-exit-node 你的节点作为出口节点，当别的机器选择你机器，所有的出口流量都从你机器流出
4. --accept-dns 使用tailscale的dns服务器(100.100.100.100)



## 网络拓扑

部署完tailscale,headscale 就可以进行内网穿透了，原理如下图。

```
            +-------------------+
            |                   |
            |  公网服务器（VPS）  |
            |                   |
            |  +-------------+  |
            |  | Headscale   |  |
            |  | (控制器)    |  |
            |  +-------------+  |
            |  +-------------+  |
            |  | Tailscale   |  |
            |  | (客户端)    |  |
            |  +-------------+  |
            |    |  100.x.x.x   |  |
            |    | Tailscale   |  |
            +----|  VPN IP     |----+
                 |             |
                 |             |  
                 |    +---------------------+
                 |    |                     |
                 |    |  Nginx 反向代理     |
                 |    |  (公网到内网转发)    |
                 |    +---------------------+
                 |             |
                 +-------------+
                               |
                               v
                       +-------------------+
                       |                   |
                       |  内网机器         |
                       |                   |
                       |  +-------------+  |
                       |  | Tailscale   |  |
                       |  | (客户端)    |  |
                       |  +-------------+  |
                       |    |  100.x.x.x  |  |
                       |    | 内网 Tailscale|
                       |    | VPN IP       |
                       +-------------------+
```

本质是使用tailscale 后，局域网，公网IP属于在同一个虚拟局域网中，公网也能连通局域网，这样就可以用nginx转发流量了。




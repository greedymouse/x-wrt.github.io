# ALG
## 1. 介绍
ALG(Application Layer Gateway),应用层网关.普通NAT实现了对UDP或TCP报文头中的的IP地址及端口转换功能,但对应用层数据载荷中的字段无能为.在许多应用层协议中,比如多媒体协议（H.323、SIP等,FTP, PPTP等,TCP/UDP载荷中带有地址或者端口信息,这些内容不能被NAT进行有效的转换,就可能导致问题.那么NAT设备内有了专门的内核模块来辅助这些协议穿越NAT.<br>
但不是所有的应用层协议都存在这样的问题, 比如L2TP经过NAT设备就没有问题.
## 2. ALG
#### 2.1 FTP ALG
##### 1. 原理
FTP在遇到NAT设备事存在的问题主要是当FTP服务器在主动模式时遇到的问题,而ftp服务器在被动模式时正常的.<br>
主动模式的问题在于:当客户端和服务端协商成功后,客户端会监听一个端口,而服务端会以端口20为源端口和对应的客户端端口发起连接,已建立数据链路.<br>
由于客户端在NAT下,因此服务端发起的连接就不能完成,所以需要NAT设备辅助完成. 
##### 2. 方案
###### (1) 辅助通过
NAT设备需要依赖模块: nf_conntrack_ftp nf_nat_ftp
###### (2) 拦截
```
1. 卸载这两个模块(nf_conntrack_ftp nf_nat_ftp)后, FTP处于被动模式就无法穿越路由器了;
2. 由于FTP在被动模式是正常的,所以如果需要进一步拦截FTP被动模式的流量,可以把TCP/21端口的数据包丢掉.
    iptables -I  FORWARD -p tcp --dport 21 -m comment --comment 'Disable-FTP-forward' -j DROP
```
##### 3. 测试
###### (1) 部署环境
部署FTP环境,可以通过docker部署ftp服务器,当然也可以通过其他方式(比如手动下载部署vsftpd)
1. docker部署FTP服务器(Linux环境):
```
$ cat compose.yaml

version: '3'

services:
  ftpd:
    container_name: ftpd
    image: fauria/vsftpd
    restart:  always
    ports:
      - "20:20/tcp"
      - "21:21/tcp"
      - 21100-21110:21100-21110/tcp
    hostname: ftpd
    environment:
      - PASV_ENABLE=NO
      - FTP_USER=user
      - FTP_PASS=123456
      - PASV_MIN_PORT=21100
      - PASV_MAX_PORT=21110
    network_mode: "host"
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"
```
```
参数:
PASV_ENABLE=NO  #主动模式
PASV_ENABLE=YES #被动模式
FTP_USER=user   #ftp用户名
FTP_PASS=123456 #ftp用户名
ftp数据目录: /home/vsftpd

启动服务:
docker-compose -f compose.yaml up -d

停止服务:
docker-compose -f compose.yaml up -v
```
2. 客户端(Windows环境或者Linux环境)
[FileZilla免费开源](https://filezilla-project.org/)
##### 4 拓展
[1. 当两次NAT碰到FTP ALG](https://yq.aliyun.com/articles/425951)

#### 2.2 PPTP ALG
##### 1. 原理
PPTP有两个数据流,一个是控制流(RFC2637定义),另外一个数据流(GRE，RFC2784).和一般的ALG不同的是(比如FTP). GRE是一个和TCP以及UDP处在同一个水平线的协议.但是和TCP/UDP之流不同的是,GRE里面不携带端口信息,所以GRE遇到NAT的时候,问题就存在了.
##### 2. 方案
###### (1) 辅助通过
NAT设备需要依赖模块: nf_conntrack_pptp nf_nat_pptp
###### (2) 拦截
卸载这两个模块(nf_conntrack_pptp nf_nat_pptp)后, PPTP数据流无法穿越路由器建立链接;
##### 3. 测试
###### (1) 部署环境
##### 4 拓展
[1. 当NAT遇到PPTP](https://www.linuxidc.com/Linux/2012-08/67884.htm)
#### 2.3 L2TP ALG
##### 1. 原理
L2TP使用纯UDP协议,所以直接穿透NAT是没有任何问题,所以不需要NAT设备做任何辅助.
##### 2. 方案
###### (1) 辅助
L2TP使用UDP协议,直接穿透NAT没有问题,所以不需要NAT设备做任何辅助.
###### (2) 拦截
目前要限制L2TP穿透NAT可以通过iptables规则限制udp目的端口:`udp/1701`
```
iptables -I FORWARD -p udp -m comment --comment 'Disable-L2TP-PassThrough' --dport 1701 -j DROP
```
##### 3. 测试
###### (1) 部署环境
[略](#)
##### 4 拓展
[略](#)
#### 2.4 IPSec ALG
##### 1. 原理
##### 2. 方案
###### (1) 辅助通过
IPsec客户端和服务端可以自己检测出是否存在NAT设备,然后自动调整策略通过NAT, 不需要NAT设备辅助.
###### (2) 拦截
目前没有办法拦截IPSec流量
##### 3. 测试
###### (1) 部署环境
##### 4 拓展
#### 2.5 SIP ALG
##### 1. 原理
SIP用于建立会话, 经常用于语言通话等等.
##### 2. 方案
###### (1) 辅助通过
通过抓包发现, 可以语音数据通过服务器代理进行转发,所以不需要辅助.
###### (2) 拦截
由于服务端端口可以改变,所以没办法拦截.
##### 3. 测试
###### (1) 部署环境
##### 4 拓展
#### 2.6 RTSP ALG
##### 1. 原理
+ RTSP是实时流媒体协议, 用于传输流媒体数据的应用层协议,是个客户端和服务端的结构.
+ 服务器端可以自行选择使用TCP或UDP来传送串流内容，它的语法和运作跟HTTP类似.
+ RTSP协议负责建立和控制媒体流的传输,发送命令和链路协商.
+ RTP协议负责传输数据流.RTSP协议在协商时客户端和服务端会定义自己监听的端口, 服务端会直接把数据发给客户端的端口.
##### 2. 方案
###### (1) 辅助通过
目前测试结果表明不需要任何辅助就可以通过.由于服务端会主动给客户端的监听端口发送数据,而且客户端在NAT设备下, 正常是不能成功的,需要NAT设备辅助, 但是分析数据包时发现在发送数据前,客户端用以为监听的端口为源端口给服务端的目的端口发送了几个数据包, 导致这条链路打通了,服务端可以直接发送数据给客户端.
###### (2) 拦截
由于不需要辅助穿透NAT, 而且协议使用TCP/UDP, 而且端口可以随意改变.没有办法准确拦截该协议的流量. 
##### 3. 测试
###### (1) 部署环境
```
1. 通过docker-compose部署rtsp服务器:
使用镜像: ullaakut/rtspatt
$ cat compose.yaml

version: '3'

services:
  rtspd:
    container_name: rtspd
    image: ullaakut/rtspatt
    restart:  always
    ports:
      - "8554:8554/tcp"
    environment:
      - RTSP_ADDRESS=0.0.0.0
      - RTSP_PORT=8554
      #- RTSP_USERNAME=test
      #- RTSP_PASSWORD=123456
      - INPUT=/tmp/test.mp4
    hostname: rtspd
    network_mode: "host"
    logging:
      driver: "json-file"
      options:
        max-size: "100m"
        max-file: "10"
    volumes:
      - ./test.mp4:/tmp/test.mp4
    # command: bash -c 'while true; do sleep 20180101; done'

启动服务:
docker-compose -f compose.yaml up -d

停止服务:
docker-compose -f compose.yaml up -v

客户端访问url: rtsp://${IP}:8554/live.sdp

2. 客户端VLC media player: 
https://www.videolan.org/
```
##### 4 拓展

# Docker Hub无法访问，DBdoctor的应对之策

近期我们收到很多小伙伴的咨询：Docker Hub无法访问，DBdoctor该如何下载安装呢?本文我们将详细给大家介绍DBdoctor的多种部署方式，以及如何快速下载以及安装部署DBdoctor。

## DBdoctor部署架构

首先我们来看下DBdoctor的部署架构，DBdoctor的安装包分为Server和Agent。目前随着国产化演进，CPU架构、OS、实例部署形态（比如docker、pod等）等不同用户存在较大的差异，但DBdoctor可以让用户屏蔽掉差异，一台数据库主机上仅需部署一个Agent，即可自动识别到主机上数据库节点的部署形态并进行探测采集（比如一台主机上既有docker部署的数据库，又有直接主机部署的，甚至有POD的，都是能做到自动识别并完成绑定纳管）。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk0bjmTCaIoXf2jaoqHQicictxKLry3vc5yiaJ6jd5aWX43ibQ3rOMiaagB7PWufyXRpZ2tjdpWUOiaj7Rg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

DBdoctor可以纳管公有云的RDS，也可以纳管自建的私有云数据库，目前各种形态都是兼容的，用户可以自行选择。

## DBdoctor下载部署方式

目前DBdoctor支持多种形态的下载部署安装方式：

### 一、主机直接一键安装（推荐）

下载地址：

- x86安装包：访问DBdoctor官网（https://dbdoctor.hisensecloud.com/h-col-133.html ），**点击免费下载即可**。

- ARM安装包：https://jhktob.oss-cn-beijing.aliyuncs.com/DBdoctorV3.2.0\_arm\_20240603.tar.gz

安装预估时间：1分钟

通过一键拉起的方式，零依赖自动在主机上部署。将下载的tar.gz安装包解压缩，进入解压后的根目录，执行./dbd -I进行DBdoctor的快速安装。例如：
```Bash
tar -zxvf DBdoctorV3.2.0_20240521.tar.gz -C ${INSTALL_PATH} 
cd ${INSTALL_PATH}
./dbd -I
```
**注**：使用./dbd -I参数执行脚本会进行单机版的一键安装。如果需要安装集群版，请参考集群安装。

关于dbd命令的更多参数说明，请参考dbd命令参数说明。

./dbd --install或-I执行安装时，可以添加选项--unlimited忽略4c8g的限制。
### 二、Docker镜像安装

由于Docker Hub国内无法访问，目前DBdoctor已将镜像仓库迁移到阿里云的ACR，同样支持docker pull直接下载安装。同时DBdoctor也支持官网直接下载docker镜像压缩包，解压导入即可进行部署安装。上述两种方式任选一种即可。

#### 1. 下载地址（任选一种）

##### a. Docker压缩包下载导入
压缩包下载地址：
- x86安装包
```
DBdoctor服务端X86安装包：https://jhktob.oss-cn-beijing.aliyuncs.com/dbdoctor-server-3.2.0\_x86.zip
DBdoctor Agent采集器X86安装包：https://jhktob.oss-cn-beijing.aliyuncs.com/dbdoctor-agent-3.2.0\_x86.zip
```
- ARM安装包
```
DBdoctor服务端ARM安装包：https://jhktob.oss-cn-beijing.aliyuncs.com/dbdoctor-server-3.2.0\_arm.zi
DBdoctor Agent采集器ARM安装包：https://jhktob.oss-cn-beijing.aliyuncs.com/dbdoctor-agent-3.2.0\_arm.zip
```
- 安装包导入步骤：

    - 需要提前安装并启动Docker服务，需要检查docker所在磁盘空间充足

    - 下载镜像文件并解压，得到dbdoctor-server-.tar

    - 导入镜像docker load -i ./dbdoctor-server-.tar
##### b. 阿里云ACR Docker镜像仓库下载地址：
```
#X86 Server/Agent下载地址
docker pull registry.cn-zhangjiakou.aliyuncs.com/dbdoctor/dbdoctor:3.2.0_x86
docker pull registry.cn-zhangjiakou.aliyuncs.com/dbdoctor/dbdoctor-agent:3.2.0_x86

#ARM Server/Agent下载地址
docker pull registry.cn-zhangjiakou.aliyuncs.com/dbdoctor/dbdoctor:3.2.0_arm
docker pull registry.cn-zhangjiakou.aliyuncs.com/dbdoctor/dbdoctor-agent:3.2.0_arm
```
#### 2. 执行docker run启动命令，需要注意Mac版与Linux版启动差异
- Linux版本：demo实例支持锁分析功能，需要挂载系统目录。
```Bash
docker run -d \
    -e HOST_IP=<hostip> \
    -e HOST_PORT=<webport> \
    -e KAFKA_HOST_PORT=<kafkaport> \
    -e OS=linux \
    -p <webport>:13000 \
    -p <kafkaport>:9092 \
    -v /proc:/host/proc:ro \
    -v /lib/modules:/lib/modules \
    -v /usr/src:/usr/src:ro \
    -v /sys/kernel/debug:/sys/kernel/debug \
    -v /<datadir>:/data \
    --privileged \
    --name dbdoctor \
    dbdoctor-server:3.2.0_x86
```
- Mac版本：demo实例不支持锁分析功能，不需要挂载系统目录。
```Bash
docker run -d \
   -e HOST_IP=<hostip> \
   -e HOST_PORT=<webport> \
   -e KAFKA_HOST_PORT=<kafkaport> \
   -e OS=mac \
   -p <webport>:13000 \
   -p <kafkaport>:9092 \
   --name dbdoctor \
   dbdoctor-server:3.2.0_x86
```
#### 3. 检查启动日志docker logs -f dbdoctor

### 三、其他方式

#### 1. 既有ARM又有X86的数据库如何部署安装？

DBdoctor的Server用户根据主机CPU架构自行选择安装包一键拉起即可。而针对Agent，自动安装的是需要数据库节点和Server节点的CPU架构一致的。当数据库节点CPU架构不一致时，可以选择单独下载适合的Agent，并进行手动安装Agent。

#### 2. K8S部署方式是否可以支持？

当然支持K8S方式部署Sever和Daemonset安装Agent，我们内部均采用此种部署架构，如果有需要可以联系我们。

## DBdoctor安装完成访问系统
DBdoctor安装完成后会打印访问地址和账号密码：
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk0bjmTCaIoXf2jaoqHQicictZvS60txpvibKF5ZtZFTKKVqSExY6tia0V1wJsq1ClPQtbCgy5XCjKP9w/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

启动成功后会打印WEB地址与账号密码：

```
WebSite:   http://xxx.xxx.xxx.xxx:13000/#/login
默认DBA用户：账号tester密码Root2023!
默认管理员：账号admin密码123456
```
DBdoctor的下载安装支持如此丰富，总有一种方式适合你~

# Docker Hub cannot be accessed, DBdoctor's response

Recently, we have received many inquiries from friends: Docker Hub cannot be accessed, how to download and install DBdoctor? In this article, we will introduce in detail the various deployment methods of DBdoctor, as well as how to quickly download, install and deploy DBdoctor.

## DBdoctor deployment architecture

First, let's take a look at the deployment architecture of DBdoctor. The installation package of DBdoctor is divided into Server and Agent. Currently, with the evolution of domestic production, there are significant differences in CPU architecture, OS, instance deployment forms (such as docker, pod, etc.) for different users. However, DBdoctor can shield users from differences. Only one Agent needs to be deployed on a database host to automatically identify the deployment form of the database node on the host and perform detection and collection (for example, if there are databases deployed by docker, directly deployed by the host, or even POD on a host, they can all be automatically identified and bound and managed).

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk0bjmTCaIoXf2jaoqHQicictxKLry3vc5yiaJ6jd5aWX43ibQ3rOMiaagB7PWufyXRpZ2tjdpWUOiaj7Rg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

DBdoctor can manage both Public Cloud RDS and self-built private cloud databases. Currently, all forms are compatible, and users can choose by themselves.

## DBdoctor download deployment method

Currently, DBdoctor supports various forms of download, deployment, and installation methods.

### 一、One-click installation directly on the host (recommended)

Download address:

- x86 installation package: visit DBdoctor official website（https://dbdoctor.hisensecloud.com/h-col-133.html ），**click to download for free .**

- ARM installation package：https://jhktob.oss-cn-beijing.aliyuncs.com/DBdoctorV3.2.0\_arm\_20240603.tar.gz

Estimated installation time: 1 minute

By pulling up with one click, zero dependency is automatically deployed on the host. Unzip the downloaded tar.gz installation package, enter the root directory after decompression, and execute./dbd-I for quick installation of DBdoctor. For example:
```Bash
tar -zxvf DBdoctorV3.2.0_20240521.tar.gz -C ${INSTALL_PATH} 
cd ${INSTALL_PATH}
./dbd -I
```
Note : Executing the script with the./dbd-I parameter will perform a one-click installation of the standalone version. If you need to install the cluster version, please refer to the cluster installation.

For more parameter descriptions of the dbd command, please refer to the dbd command parameter description.

When installing with./dbd --install or -I, you can add the option --unlimited to ignore the limitations of 4c8g.
### 二、Docker mirroring installation

Due to the inaccessibility of Docker Hub in China, DBdoctor has migrated the mirroring repository to Alibaba Cloud Ali Cloud Aliyun's ACR, which also supports direct download and installation of docker pull. At the same time, DBdoctor also supports direct download of docker mirroring Compressed Packet from the official website, and can be deployed and installed by decompressing and importing. Either of the above two methods can be selected.

#### 1. Download address (choose one)

##### a. Docker Compressed Packet Download Import
Compressed Packet download link:
- x86安装包
```
DBdoctor server level X86 installation package:https://jhktob.oss-cn-beijing.aliyuncs.com/dbdoctor-server-3.2.0\_x86.zip

DBDoctor Agent Collector X86 Installation Package：https://jhktob.oss-cn-beijing.aliyuncs.com/dbdoctor-agent-3.2.0\_x86.zip
```
- ARM installation package
```
DBdoctor server level ARM installation package：https://jhktob.oss-cn-beijing.aliyuncs.com/dbdoctor-server-3.2.0\_arm.zi

DBDoctor Agent collector ARM installation package：https://jhktob.oss-cn-beijing.aliyuncs.com/dbdoctor-agent-3.2.0\_arm.zip
```
- Installation package import steps：

    - You need to install and start the Docker service in advance, and check that the disk space where Docker is located is sufficient

    - Download the mirroring file and extract it to get dbdoctor-server-.tar

    - import mirroring docker load -i./dbdoctor-server-.tar
##### b. Alibaba Cloud Ali Cloud Aliyun ACR Docker mirroring repository download link：
```
#X86 Server/Agent Download Address
docker pull registry.cn-zhangjiakou.aliyuncs.com/dbdoctor/dbdoctor:3.2.0_x86
docker pull registry.cn-zhangjiakou.aliyuncs.com/dbdoctor/dbdoctor-agent:3.2.0_x86

#ARM Server/Agent Download Address
docker pull registry.cn-zhangjiakou.aliyuncs.com/dbdoctor/dbdoctor:3.2.0_arm
docker pull registry.cn-zhangjiakou.aliyuncs.com/dbdoctor/dbdoctor-agent:3.2.0_arm
```
#### 2. Execute the docker run startup command, pay attention to the startup differences between Mac and Linux versions
-  Linux version: demo instance supports lock analysis function, need to mount system directory.
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
- Mac version: The demo instance does not support lock analysis function and does not need to mount the system directory.
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
#### 3. Check startup logs docker logs -f dbdoctor

### 三、Three, other ways

#### 1. How to deploy and install a database with both ARM and X86?

DBdoctor's Server users can choose the installation package according to the host CPU architecture and pull it up with one click. For Agent, the automatic installation requires the CPU architecture of the database node and the Server node to be consistent. When the CPU architecture of the database node is inconsistent, you can choose to download the appropriate Agent separately and manually install the Agent.

#### 2. Can the K8S deployment method be supported?

Of course, we support K8S deployment of Server and Daemonset installation agents. We use this deployment architecture internally. If necessary, please contact us.

## DBdoctor installation completed to access the system
After the installation of DBdoctor is completed, the access address and account password will be printed.

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk0bjmTCaIoXf2jaoqHQicictZvS60txpvibKF5ZtZFTKKVqSExY6tia0V1wJsq1ClPQtbCgy5XCjKP9w/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

After successful startup, the WEB address and account password will be printed.

```
WebSite:   http://xxx.xxx.xxx.xxx:13000/#/login
Default DBA user: Account tester password Root2023!
Default administrator: account admin password 123456
```
DBdoctor's download and installation support is so rich, there is always a way that suits you~

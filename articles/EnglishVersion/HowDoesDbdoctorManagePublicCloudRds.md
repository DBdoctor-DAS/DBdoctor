# DBdoctor如何纳管公有云RDS

最近很多小伙伴问到DBdoctor能不能纳管公有云上的RDS，答案当然是可以的。下面就以阿里云为例，介绍下DBdoctor自定义扩展RDS模块并纳管。

## 1）DBdoctor采集支持三种数据源接入
DBdoctor采集支持三种数据源来满足不同场景用户接入：

- DBdoctor agent数据采集：需要在数据库节点的主机上进行部署，支持页面配置自动安装，具备更多深度数据指标采集。

- Prometheus数据接入：如果用户已有数据采集工具且提供Promethus格式的监控数据，DBdoctor可无缝对接promethus，无需重复部署采集工具。

- 自定义第三方扩展：DBdoctor支持自定义第三方服务接口，可使用Python脚本自定义适配对接，比如对接公有云的RDS OpenAPI。
## 2）DBdoctor 一分钟快速部署，零依赖
```
#环境要求：推荐4c8g
#下载安装包：https://www.hisensecloud.com/h-col-133.html
#官方安装文档：https://dbdoctor.hisensecloud.com/modules/dbDoctor/mdPreview/index.html?readme=help#/mdManageDocument/6.1.1-Quick-Install
```
## 3）DBdoctor快速自定义扩展兼容第三方监控服务
搭建好DBdoctor的Server端服务后，需要安装阿里云的OpenAPI的Python SDK模块。

#### a）安装阿里云SDK

##### 1、cd到DBdoctor安装目录，并执行
```Bash
chroot dbdoctor-3.1.2
```
##### 2、执行如下命令安装阿里云SDK（建议使用国内的镜像源）
```Bash
pip3 install --upgrade pip -i https://mirrors.aliyun.com/pypi/simple
pip3 install alibabacloud_rds20140815==5.0.0 -i https://mirrors.aliyun.com/pypi/simple
pip3 install flask -i https://mirrors.aliyun.com/pypi/simple
```
#### b）创建对接阿里云RDS OpenAPI脚本模版

1. 使用admin用户（默认密码：123456）登录DBdoctor，点击数据采集管理->模板配置->创建模板


![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk6voTsqLmkFxnX1TgHjiaEKuIZRNqeNUGlccRKvmErBY2OV3rQicvTngQedSt8dZTPNKcVuxz1DEjA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

2. DBdoctor提供第三方监控扩展模块，可以自定义编写脚本。**点击创建模板后模板类型选择自定义**，同时填写模板名称、模板描述、将下面RDS OpenAPI对接脚本复制到Python 代码输入框，点击提交即可。

下面是扩展阿里云RDS模版的Python代码：

```
# 阿里云RDS模板Python代码下载地址：
https://jhktob.oss-cn-beijing.aliyuncs.com/aliyun.py
# 注意事项：模板代码中的access_key_id与access_key_secret需替换成真实的aksk。
```
阿里云的aksk需要控制台开通下面两个接口的权限：
```
# 需要阿里云控制台授权的OpenAPI接口
# 接口一：
DescribeDBInstanceAttributeRequest
# 接口二
DescribeDBInstancePerformanceRequest
```
### 4）纳管阿里云RDS
1. 退出admin账户，使用tester账户（默认密码：Root2023!）登录

2. 点击实例纳管，其中数据库采集方式选择上面新建的模板。第三方参数配置，key值为instanceId，value为待纳管实例在阿里云的实例ID，最后点击提交。

![实例纳管](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk7Og3xibBEWm2RM9YfMopwg1qLH9NVTmbDSttZLficM1qtT2jDicdo2ONeArGTJogzk34NlDYmUoqNw/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

3. 最后打开性能分析即可
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk7Og3xibBEWm2RM9YfMopwgD1PDRFQ9bnUha3l9bVEnEiaUKwRm0g28ordMqsL7WyicRNPEGiaiakcFEg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 5）特色功能介绍

#### a）性能洞察
1. 实例列表中找到已纳管的RDS数据库，点击性能洞察开关按钮即可开启分析
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk7Og3xibBEWm2RM9YfMopwgD1PDRFQ9bnUha3l9bVEnEiaUKwRm0g28ordMqsL7WyicRNPEGiaiakcFEg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)
2. 点击实例透视RDS数据库的性能
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk7Og3xibBEWm2RM9YfMopwgQJ0X4kONy5Bp0Bd53zjiaJEHnJ9KvlVsUeJKq5GptPFsAh8A11vuicmQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

性能洞察功能可将硬件资源与数据库的指标性能数据，通过时间轴关联起来并进行多维度分析。比如上面的**CPU异常的Case**我们只需要**1步走即可找到问题根因SQL，同时可以还原异常时间区间详细现场。**

- 自动根因识别：cpu资源指标发生抖动，自动框选异常区间并提示根因SQL。

- 详细现场还原：
    - CPU异常时间区间框选
    - 异常时间区间在AAS模块中我们能看到数据库的活跃会话数在这一时间区间内远超Max vCPU水位线，说明存在性能瓶颈。
    - 可以看到性能瓶颈的事件是ACTIVE（即蓝颜色事件），而这一颜色在AAS趋势图中占的面积最大。
    - 基于这个面积最大颜色的事件，我们能找到绿色颜色的SQL为第一条，即导致CPU飙高的根因SQL。点击展开这个SQL可以展示这个SQL的最差样本，点击执行计划发现扫描全表扫描行，对SQL进行添加索引即可。
#### b）SQL审核
DBdoctor除了支持业界主流的基于规则审核的同时，还创新性的实现了覆盖性能诊断的SQL审核，能够基于线上数据模型，在发布上线前快速、精准评估SQL性能问题并给出优化建议，提前规避线上故障的发生。
![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk7Og3xibBEWm2RM9YfMopwg0lBAOpaRhG4p2QjWDyQKCzUjjsv8mq5pxWdesic768mhTTaIsRMgExg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

业界独家上线前评估SQL性能的审核，原理介绍见：史上最强的SQL审核工具！

OMG，纳管阿里云RDS并进行数据库性能诊断与SQL审核竟如此简单!而且对于 RDS，DBdoctor还支持实例巡检、基础监控、根因诊断、索引推荐等等功能。这么实用又免费的工具，不用真的亏！


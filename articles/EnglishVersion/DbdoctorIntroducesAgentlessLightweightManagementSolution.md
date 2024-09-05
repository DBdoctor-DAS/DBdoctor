# DBdoctor推出无Agent轻量级纳管解决方案

## 背景
在数字化时代，数据库作为信息系统的核心，其安全性、稳定性与高效性至关重要，对于业务敏感且数据保密性要求高的企业而言，在数据库权限管理和安全流程上有着更为严苛的要求。当这些用户在面临数据库性能问题时，往往无法使用那些通过部署Agent来采集数据及执行命令的辅助工具，来快速发现并解决问题，因此这让他们倍感困扰！

针对上述难题，DBdoctor现已推出无Agent轻量级纳管解决方案，无需安装Agent,即可快速纳管实例！

## DBdoctor推出无Agent轻量级纳管解决方案

DBdoctor无Agent轻量级纳管方式，可有效简化部署过程，快速对数据库进行性能诊断。

### 方案优势：

- 轻量部署，快速纳管实例

- 无需安装Agent，不再依赖kernel版本、cpu架构

- 纳管后可使用：**SQL审核、索引推荐，实例巡检、性能洞察、基础监控**功能

### 实例纳管方式：

数据库基本信息填写校验完成后，数据采集方式选择【无Agent】，点击提交即可完成无Agent实例纳管。选择该方式后，将不会再到目标机器上进行Agent安装。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZlsFicCwVJTibCsM3ic5SoNict1A6SHyaPJSZ9zvOKhbsfHTnE1wmtvowqSQ9FID9jzFeVFsCSAD2jTcA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

**注**：因CPU、diskIO、MEM、diskSize等指标依赖Agent采集，因此性能洞察、实例巡检、基础监控等部分功能将受限。

## 无Agent纳管可体验哪些功能？

### 1. 全量SQL审核功能
- 解锁人工审核、慢SQL审核、OpenAPI审核

- 解锁索引推荐功能

- 如下图所示，我们可以看到针对用户提交的批量SQL审核，可进行规则审核及性能审核

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpIN0MFKZdG8xBOWgzSvOg1jLXmatEq05Y99rpnWWrpKuEkd38928m5g/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpv1lGiajvb0hbTUJ4BrF9ZMaXfhwIO4mInjUnOqTJMvSaazRZNQz2FjQ/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 2.实例巡检功能

- 可正常使用巡检功能，我们通过点击【立即巡检】按钮即可触发巡检功能，等待巡检报告，可查询巡检报告结果。

- 可体验巡检总览、监控状况概要、配置问题、资源问题、性能趋势等功能。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpgibhTTLnicCaSUrtQnibx0jI7T5Q8Dop9javsBABL9CqwRPSicZTc5xE8Q/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 3. 性能洞察功能

通过AAS数据及慢SQL，我们同样可以分析业务问题找到根因SQL，如图所示，业务在某一时刻发现卡顿，通过DBdoctor，我们可以看到 AAS凸起部分获取根因SQL。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEpUbbUiaicibbUp7SrDXeqkN49rZyAqa2wKSsAm2tzZQHn5crSiaavaWOzNA/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 4. 基础监控功能

无Agent纳管实例基础监控可正常使用，因 CPU、diskIO、MEM、diskSize、存储空间相关指标为Agent采集，相关指标无法展示，其他功能不受影响。

![](https://mmbiz.qpic.cn/mmbiz_png/dFRFrFfpIZk8VvtXffcegrISvVpVCibEp7htp78SNNiaxh2U4QFRdMbAgOmXDQYf1Jz00ialqKbKUFe7kw8uSbrQg/640?wx_fmt=png&from=appmsg&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

## 总结

DBdoctor推出的无Agent数据采集方式在简化部署、增强兼容性、提升安全性及快速集成扩展等方面具有显著优势，能够为金融等对数据安全有特殊要求的用户提供更高效、安全、便捷的数据库性能诊断服务。通过无Agent纳管实例，可方便小伙伴快速体验SQL审核、性能洞察等功能，但同时因采集限制，锁透视、审计日志、根因分析等部分功能将受限，因此想体验DBdoctor全量功能，还是建议小伙伴们通过Agent方式进行实例纳管。
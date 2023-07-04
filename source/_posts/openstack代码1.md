---
title: Nova源码阅读（一） openstack简介
tags: openstack
abbrlink: ecb3c3ec
date: 2021-08-30 14:20:07
---

内容主要摘自于《Openstack设计与实现》。

## openstack与AWS

与AWS相比，openstack只是处于一个追随者的位置。

<!-- more -->

### AWS架构

AWS由五层组成，自下而上包括AWS全球基础架构，基础服务，应用平台服务，管理和用户应用程序。

AWS提供六类主要服务：数据库，存储和内容分发，分析，计算和网络，部署管理，应用服务。

- 数据库：NoSQL数据库服务DynamoDB，关系数据库服务RDS，缓存和数据仓库服务Redshift
- 存储和内容分发：简单存储服务S3，块存储服务EBS，Amazon云前端Cloud Front，Amazon Glacier，AWS存储网关Storage Gateway。
- 分析：用于大数据的EMR，用于大规模实时流数据处理Kinesis，数据管道Data Pipeline
- 计算和网络服务：负责虚拟机调度和管理EC2，保证企业在公有云上搭建安全私有云的虚拟私有云服务VPC，负载均衡ELB，虚拟桌面管理服务WorkSpaces，计算资源自动扩容缩容的Auto Scaling，为企业定制的专属网络连接DirectConnect，高可靠性且可扩展的域名系统web服务Route 53
- 部署管理：建立和管理AWS资源CloudFormation，监控CloudWatch，轻松部署Web应用和服务的Elastic Beanstalk，验证访问管理IAM，日志管理CloudTrail，为运维人员配备的应用管理服务OpsWorks和安全服务CloudHSM
- 应用服务：云搜索CloudSearch，流媒体转码Elastic Transcoder，简单邮件服务SES，简单队列服务SQS，简单工作流服务SWF，应用程序流AppStream


### AWS与Openstack相对应的项目

|      AWS模块      |         Openstack模块          |
| :---------------: | :----------------------------: |
|        EC2        |              Nova              |
|        S3         |             Swift              |
|        EBS        |             Cinder             |
|        IAM        |            Keystone            |
|    CloudWatch     |           Ceilometer           |
|  CloudFormation   |              Heat              |
| 支持RDS, DynamoDB | 支持MySQL，PostgreSQL，MongoDB |

## openstack 架构

{% asset_img  openstack标准架构.jpeg 图1 openstack结构%}

OpenStack曾经包含7个核心组件：

| 组件名称 |                             作用                             |
| :------: | :----------------------------------------------------------: |
|   Nova   |                        提供虚拟机服务                        |
|  Swift   |                     存储和检索对象/文件                      |
| KeyStone |             身份认证和授权，跟踪用户及他们的权限             |
| Horizon  |                           提供界面                           |
|  Cinder  |                          块存储服务                          |
| Neutron  | 网络连接服务，允许用户创建自己的虚拟网络并连接各种网络设备接口 |
|  Glance  |               虚拟机镜像的存储，查询和检索服务               |




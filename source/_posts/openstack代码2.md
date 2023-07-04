---
title: Nova源码阅读（二） openstack通用技术
tags: openstack
abbrlink: 7952953b
date: 2021-08-31 15:37:28
---



内容主要摘自于《Openstack设计与实现》。

<!-- more -->

## 消息总线

Openstack遵循这样的原则：项目之间通过**RESTful API**进行通信，项目内部的不同服务进程之间的通信，则必须通过**消息总线**。oslo.messaging 库实现了以下两种方式来完成项目内部的不同服务进程之间的通信。

* 远程过程调用（RPC）。通过RPC，一个服务进程可以调用其他远程服务进程的方法，并且有两种调用方式：同步和异步。
* 事件通知（Event Notification）。某个服务进程可以把事件通知发送到消息总线上，该消息总线上的所有对此类事件感兴趣的服务进程，都可以获得此事件通知并进行进一步的处理，但是处理的结果并不会返回给事件的发送者。

### AMQP(advanced message queuing protocol)

在Openstack支持的消息总线类型中，大部分都是基于AMQP的。AMQP是一个异步消息传递所使用的开放的应用层协议，主要包括信息的导向，队列，路由，可靠性和安全性。oslo.messaging中支持的AMQP主要包括两个版本，AMQP 0.9.1和AMQP 1.0，两个版本有较大区别。

#### AMQP 0.9.1

{% asset_img  AMQP.PNG 图1 AMQP结构 %}

当不同的消息由生产者（Publisher application）发送到broker时，会根据不同的条件把消息传递给不同的消费者（consumer）。如果消费者无法接收到信息或者接受信息不够快时，会把消息缓存在内存和磁盘上。

生产者将消息发送给Exchange，并由Exchange来决定消息的路由，即消息发送给哪个Queue，然后消费者从Queue中取出消息，进行处理。

每一个消息都有一个routing key，每一个queue有一个binding key，Exchange在收到消息时，会将routing key与每一个queue的binding key进行匹配，匹配成功，则会将信息转发到对应的queue中。

转发的类型有三种：

* direct：routing key与binding key完全一致，不支持通配符；

* topic：支持通配符；

* fanout：忽略routing key与binding key。消息转发到所有queue中。

#### RabbitMQ

RabbitMQ是一个实现了AMQP的消息中间件服务，支持多种语言的客户端开发，支持用户自定义插件开发的框架及多种插件。

#### AMQP 1.0

与AMQP 0.9.1相比，1.0协议更加灵活和复杂。1.0实现了一种消息路由模式：位于调用者和服务器之间的不再是单节点的broker，而是一群互相连接的消息路由组成的路由网；路由不具备queue，没有存储消息的能力，作用只是传递消息；路由节点之间通过使用tcp连接进行通信；调用者可以通过tcp连接连到路由网中的某个路由，从而接入路由网。

## SQLAlchemy与数据库

{% asset_img  SQLAlchemy.PNG 图2 SQLAlchemy结构%}

SQLAlchemy 主要包括两个部分：SQLAlchemy Core和SQLAlchemy ORM。SQLAlchemy core包括SQL语言表达式，数据引擎，连接池等。所有的实现都是以连接不同类型的后台数据库，提交查询和更新SQL请求，定义数据库数据类型和定义schema等为目的。SQLAlchemy ORM提供数据映射模式，即把程序语言的对象数据映射成数据库的关系数据，或者把关系数据映射为对象数据。另外SQLAlchemy对象关系映射是一个可选模块，开发人员可以完全不使用任何对象模型，直接使用SQLAlchemy操作数据。

## RESTful 与 WSGI

### RESTful

参考文献：

1. https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm
2. http://restful.p2hp.com/

REST（representational state transfer，表述性状态转移）。从RESTful角度上看，网络中的所有东西都是资源，每个资源都对应于一个特定的URI，并用它进行标识。资源有多种具体的表现形式，即资源的表述。URI仅代表资源的实体，不能代表其表现形式。

REST的指导原则

1. **客户端 - 服务器** 。
2. **无状态**。从客户端到服务器的每个请求都必须包含理解请求所需的所有信息，并且不能利用服务器上任何存储的上下文。因此，会话状态完全保留在客户端上。
3. **可缓存** 。缓存约束要求将对请求的响应中的数据隐式或显式标记为可缓存或不可缓存。如果响应是可缓存的，则客户端缓存有权重用该响应数据以用于以后的等效请求。
4. **统一接口** 。
5. **分层系统**。 分层系统风格允许通过约束组件行为来使体系结构由分层层组成，这样每个组件都不能“看到”超出与它们交互的直接层。
6. **按需编码（可选）**。 REST允许通过以小程序或脚本的形式下载和执行代码来扩展客户端功能。这通过减少预先实现所需的功能数量来简化客户端。

### WSGI（Web service gateway interface）

RESTful只是设计风格而不是标准，WEB服务通常使用基于HTTP的符合RESTful的API，而WSGI则是python语言中所定义的WEB服务器和WEB应用程序或框架之间的通用接口标准。

当处理一个WSGI请求时，服务端会为应用端提供上下文信息和一个回调函数，应用端在处理完请求后，会使用服务端所提供的回调函数返回对应请求的响应。

WSGI将WEB组件分成三类：WEB服务器，WEB中间件，WEB应用程序。WEB服务器用于接收HTTP请求，封装一系列环境变量，按照WSGI接口标准调用注册的WSGI应用程序，然后将响应返回给客户端。

WSGI应用程序是一个可被调用的Python对象，只接受两个参数，environ和start_response。

* environ指向一个python字典，要求里面至少包含一些在CGI中定义的变量以及至少包含其他7个WSGI所定义的环境变量。WSGI应用程序会从environ字典中获取相应的请求及其执行上下文中的所有信息。

* start_response指向一个回调函数：

  `start_response(status, response_headers,exe_info=None)`

  status是一个响应状态的字符串；response_headers是一个包含了（header_name,header_value）元组的列表，分别表示HTTP相应中的HTTP头和内容；exe_info一般在错误时使用，用来让浏览器显示相关错误信息。

  回调函数返回形如write(data)的可被调用的对象。这个对象是为了兼容现有的一些特殊框架而设计的，一般不使用。

#### Paste 文件

  Openstack使用Paste的Deploy组件完成WSGI Server和WSGI Application的构建，每个项目源码的etc目录下都有一个Paste文件，如Nova的etc/nova/api-paste.ini，在部署时，这些文件被拷贝到系统/etc/\<project\>/目录下。

Paste配置文件分为多个section，每个section以type:name命名。

* type = composite

  这个section会把URL请求发到对应的应用程序中，并由use指定具体的分发方式。

* type = app

  一个app就是一个具体的WSGI 应用程序，这个app对应的python代码可以由use来指定。另一种指定方法是明确指定对应的python代码，这时必须给出代码应该符合的格式。

* type = filter-app

  在接收到一个请求后，会先调用fliter-app中的use所指定的app进行过滤，如果请求没有被过滤，就会被转发到next所指定的app上进行下一步的处理。

* type = filter

  跟上一个的区别是没有next。

* type = pipeline

  由一系列filter组成，filter链条的末尾是一个app。pipeline类型对filter-app做了简化。

#### WebOb

WebOb通过对WSGI的请求与相应进行封装，可以简化WSGI应用的编写。

WebOb重要的对象：

* webob.Request。对WSGI请求的参数environ进行封装；

* webob.Response。包含了标准WSGI响应的所有要素；

* webob.exc。针对HTTP错误代码进行封装。

* webob.dec.wsgify。以便可以不使用原始的WSGI参数和返回格式，而全部使用WebOb替代。

#### Pecan

Paste组合框架的REATful API代码过于臃肿，导致项目的可维护性差。一些新项目使用Pecan框架实现RESTful API。Pecan是一个轻量级的WSGI网络框架。其设计思想不是解决Web世界的所有问题，而是主要集中于对象路由和RESTful支持上，并不提供对话和数据库支持。  

## Eventlet 与 AsyncIO

Openstack中的绝大多数项目采用协程模型。协程，是一种比线程更加轻量级的存在，协程拥有自己独立的栈和局部变量，同时与其他协程共享全局变量。

协程与线程的区别是：多个线程可以同时执行，但是同一时间只能有一个协程运行；线程的执行由操作系统控制，协程的执行顺序由程序自己决定。由于协程无需考虑很多有关锁的问题，因此开发和调试比较简单。

### Eventlet

协程来实现并发，将协程称为GreenTread，所谓的并发就是建立多个GreenTread，并对其进行管理。

### AsyncIO

Eventlet具有局部性，如不支持Python3，PyPy，Jython等，只支持CPython。目前Openstack正考虑用AsyncIO替代Eventlet。


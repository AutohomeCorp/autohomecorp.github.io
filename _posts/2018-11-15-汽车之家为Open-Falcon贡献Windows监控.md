---
layout: post
title: 汽车之家为Open-Falcon贡献Windows监控
category: articles
tags: [汽车之家,监控,open-falcon,开源]
author: wangjinlong
comments: true

---

# 前言
小米Open-Falcon监控系统自2015年开源以来，以其灵活的数据采集，良好的性能表现，高效的告警策略等特性，赢得的众多互联网公司的青睐。
汽车之家也一直关注着Open-Falcon的发展，系统平台团队通过对Open-Falcon的二次开发，打造了汽车之家的监控系统。这套系统负责了汽车之家所有服务器基础监控，URL监控，日志监控等重要功能。作为公司基础系统，稳定高效的支撑了近万台服务器的监控，告警工作。

# 设计

### 初衷

汽车之家除Linux服务器外，还有很多业务运行在Windows机器上，所以对Windows服务器基础监控，IIS，SQL Server等Windows服务的监控也非常重要。但是Open-Falcon未全面覆盖Windows系统，没有官方的Windows Agent去做数据的采集。社区中开源的脚本都是通过计划任务的方式采集。而我们希望的是Open-Falcon在Windows下的Agent采集的逻辑和架构与Linux下保持一致，方便监控平台管理，控制Agent。

### 目标

我们的设计目标有以下几点：

1. 可以服务的形式运行在Windows服务器上，不用配置计划任务
2. 支持采集Windows服务器基础监控项
3. 支持采集IIS，SQL Server的监控项采集
4. 提供和Linux Agent一样的push数据接口，支持第三方push数据
5. 与Linux Agent其他功能保持一致

基于以上几点我们自研了之家的`Open-Falcon Windows Agent`。

### 实现

#### 代码架构

![image](/images/windows_agent/arch.png)

`Windows-Agent`的代码架构如上图所示。程序启动后，会启动5个线程。每个线程都会按照配置好的时间间隔定时采集所需信息。

* `basic thread` 基础监控项采集线程，通过`psutil`这个跨平台的库，可以轻松获取操作系统进程和系统利用率等信息。

* `IIS thread` IIS数据采集线程，通过`winstats`这个库，定时的采集IIS站点的连接数，IIS站点的cpu使用率等数据。

* `SQL Server thread` SQL server数据采集线程，同样通过`winstats`, 获取到SQL Server内存和I/O相关数据。

* `status thread` Agent自身状态线程。这一点和`Linux Agent`的功能一样， 定时向`Hearbeat Server`汇报自己Agent的状态。这样在我们的监控平台上就可以向管理Linux服务器一样的管理这些Windows服务器。

* `HTTP` HTTP线程会开启一个HTTP服务提供push接口，和Linux Agent一样，用户可以选择通过该push接口，把自定义的数据push给Agent。方便第三方数据的接入。

#### 数据的传输

Open-Falcon Linux下的Agent启动之后，会和transfer组件建立长连接，通过`Transfer.Update`这个RPC调用，把Agent采集到的监控数据传输给transfer，后面的事情就全部交由Open-Falcon处理。Agent自身状态的汇报也同样方式，通过`Agent.ReportStatus`这个RPC调用和`Hearbeat Server`交互，上报自身状态。在Windows下，我们要采用同样的方式和`transfer`组件，`Hearbeat Server`组件进行数据的传输，不同的是，Linux下的Agent是golang实现，可以方便的使用golang原生的JSONRpc处理RPC调用，而我们Windows下的Agent使用python开发， 所以我们自己实现了jsonrpc的client，来模拟Linux下的处理。保证我们的Agent行为和Linux下的Agent一致

#### 如何变身Windows服务

Windows Agent通过`pypiwin32`这个库，把python代码变成了服务安装到了Windows服务器上。这个库怎么用呢？Demo如下：

```python
class AppServerSvc(win32serviceutil.ServiceFramework):
    _svc_name_ = "OpenFalconAgent"
    _svc_display_name_ = "Open-Falcon Windows Agent"

    def __init__(self, args):
        win32serviceutil.ServiceFramework.__init__(self, args)
        self.hWaitStop = win32event.CreateEvent(None, 0, 0, None)
        socket.setdefaulttimeout(60)
        self.isAlive = True

    def SvcStop(self):
        self.ReportServiceStatus(win32service.SERVICE_STOP_PENDING)
        win32event.SetEvent(self.hWaitStop)
        self.isAlive = False

    def SvcDoRun(self):
        servicemanager.LogMsg(servicemanager.EVENTLOG_INFORMATION_TYPE,
                              servicemanager.PYS_SERVICE_STARTED,
                              (self._svc_name_, ''))
        self.isAlive = True
        #do somethings
        self.main() 
        
if __name__ == '__main__':
    win32serviceutil.HandleCommandLine(AppServerSvc)
```
首先要继承`win32serviceutil.ServiceFramework`这个类，然后分别实现构造方法，停止运行方法`SvcStop`, 以及启动方法`SvcStop`。最后在主方法中调用`win32serviceutil.HandleCommandLine(AppServerSvc)`。
就可以通过`python agent.py install`安装服务，`python agent.py start`启动服务，有兴趣的同学不妨可以自己试试。

#### 配置文件
Windows Agent的配置文件也和Linux Agent一下保持一致，如果你熟悉了Linux下的配置，甚至可以直接copy到Windows服务器下。具体的配置解释如下

##### Basic config

| key | type | descript|
|-----|------|----|
| debug | bool | whether in debug mode or not|
| hostname | string | the same as OpenFalcon Linux Agent|
| ip | string | ip address|
| heartbeat | dict | details in the later of this file |
| transfer | dict | details in the later of this file |
| http | dict | details in the later of this file |
| collector | dict | details in the later of this file |
| ignore | array | the metrics you wanna ignore |

##### Heartbeat config

| key | type | descript|
|-----|------|----|
| enabled | bool | whether enable send heartbeat to hbs|
| addr | string | ip adrress of hbs|
| interval | int | intervals between two heartbeat report|
| timeout | int | timeout |

##### Transfer config

| key | type | descript|
|-----|------|----|
| enabled | bool | whether enable send data to transfer|
| addr | dict of string | ip adrresses of all transfer |
| interval | int | intervals between two heartbeat report|
| timeout | int | timeout |

##### Http config 
| key | type | descript|
|-----|------|----|
| enabled | bool | whether enable http api|
| listen | string | the port server listened on|

# 安装

* 克隆下代码库`git clone git@github.com:AutohomeRadar/Windows-Agent.git`
* 安装依赖`pip install -r requirements.txt`
* 修改配置文件`cfg.json`中的内容为自己的正确地址
* `python agent.py install`安装Agent服务
* `python agent.py start`启动agnet

# 实战
目前Windows Agent运行在汽车之家上千台Windows服务器下2年多时间，始终保持了稳定，可靠的数据采集，同时对资源的消耗也非常小。

下图为Agent作为服务运行

![image](/images/windows_agent/server.png)


下图为Agent进程的消耗，由于我们内部的Agent监控项要比开源的版本多，所以内存占用大概有30M左右，开源版本的内存占用会小于这个数值

![image](/images/windows_agent/server2.png)

下图为采集到的IIS站点的cpu使用率监控数据

![image](/images/windows_agent/iis.png)

下图为SQL Server采集监控数据

![image](/images/windows_agent/sql.png)
# 开源

在公司的支持下，我们将代码以Apache许可证开源。目前`Windows Agent`组件已经被Open-Falcon社区收录，为更多Windows用户提供支持。

![image](/images/windows_agent/open.png)

相关文档以及代码参见`https://github.com/AutohomeRadar/Windows-Agent/`

# 后续计划
我们计划下一步为`Windows Agent`加入更多的特性。例如对插件的支持，添加更丰富的监控项等。同时，汽车之家对Open-Falcon还做了很多的二次开发，比如告警的升级机制，多种维度的告警收敛，URL监控，网络监控等，并且已经应用到生产环境当中。以后我们也会把通用的组件开源，回馈社区。
同时也感谢小米，感谢Open-Falcon社区。
更多精彩技术文章，欢迎大家访问汽车之家运维团队博客`http://autohomeops.corpautohome.com`
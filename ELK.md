# 基于ELK架构的日志服务

我所在的公司随着业务越来越多, 项目也越来越多, 随之而来的就是日志越来越多. 一方面为了更好的收集用户的数据, 日志收集可以说是大数据的基石,也为公司未来挖掘用户数据打下基础,另一方面也为了更好的给开发提供线上日志搜索服务,帮助开发定位问题. 我负责给公司搭建基于ELK架构一个高可用性,高可靠性和可扩展性的日志服务日志服务系统.

## 平台简介

先简单介绍一下平台用到那些组建:

* Apache Flume: flume 是一个高可用的分布式系统, 能够高效的收集,聚合数据, 将来自不同源的数据投递到不同的数据容器中. Flume的数据单元是Event, Event将贯穿整个数据流, 从Source到Channel最终到Sink.


    * source: 实时读取日志封装成event批量放入channel, event是数据单元, 带有一个可选的消息头header
    * channel: 可以理解成日志数据的缓冲区
    * sink: 从channel里拿到event,然后在原始日志前面加上header, 批量发送到kafka, 我们可以对header做一些自定义:
    ```
    header: {
        ip,
        appName, // 应用名称
        logType // 日志类型, 访问日志, tomcat日志等
    }
    ```
如何自定义header呢, flume提供拦截器[Flume-interceptors](http://flume.apache.org/FlumeUserGuide.html#flume-interceptors), 这边关于如何自定义拦截器的blog写的也比较好[flume-getting-started-with-interceptors](https://hadoopi.wordpress.com/2014/06/11/flume-getting-started-with-interceptors/).
如何做到Flume服务的高可用, 所有的Agent服务在supervise的方式下启动, 如果进程死掉会被系统立即重启,继续提供服务, 其次就是做对Agent服务的监控, 当Agent死掉立刻报警, 常用的监控方式有monitor.

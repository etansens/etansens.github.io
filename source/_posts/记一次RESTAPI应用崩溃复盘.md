---
title: 记一次RESTAPI应用崩溃复盘
date: 2019-11-01 16:44:09
categories:
- Web后端
tags: 
- java
- spring
- jdbc
- tomcat
---
记一次RESTAPI应用崩溃复盘，第一次写博，没有代码样例，没有日志截图，全凭想象。
同一个世界，不同的场景，本文仅代表个人经验。
<!--more-->
# 构架环境
centos6.5
lvs
jdk1.8.0_91
tomcat9.0.20
spring-boot2.1.3
oracle11.2.0.4
ojdbc6
JAVA_OPTS="$JAVA_OPTS 
-Xms8g -Xmx8g -Xss256k 
-XX:+UseParNewGC 
-XX:+UseConcMarkSweepGC 
-XX:CMSInitiatingOccupancyFraction=75 
-XX:+UseCMSInitiatingOccupancyOnly 
-XX:+DisableExplicitGC 
-XX:+HeapDumpOnOutOfMemoryError"

# 工作流程
一台API提供HTTP服务，后端通过lvs负载访问两台oracle数据库

# 场景还原
每隔5分钟高频访问,5kQPS峰值持续1分钟；每隔一个小时有大查询持续2分钟
监控发现大量502错误，tomcat进程CPU使用率1000%，10分钟内服务僵死

# 调整过程
0、查看日志，发现内存溢出异常

1、首先观察到的是内存问题，jstat发现old区内存爆满，持续FULL GC；增大内存到16G并调整为G1GC策略;

2、运行1小时后发现内存还是爆了，排查发现大量线程内存不能释放，
   查看tomcat参数，maxThreads="32000"怀疑该参数过大，通过几轮调整测试发现每线程占用20M-25M内存
   决定设置maxThreads="96"再观察;

3、运行一天后服务在某个时间点僵死，再查看日志，发现大量无法获取数据库连接错误，排查代码没有发现泄漏
   通过TCP查看物理连接情况，发现实际连接较少；再观察数据库端session数量，发现session数量已经到达max值，
   大量idle连接等待超时；
   此时重点排查数据库连接过程：APP->LVS->DB；首先绕过LVS直连DB，APP恢复正常，问题得到解决！
   再排查LVS参数：persistence_timeout 60，ipvsadm --set 120 5 60；发现是LVS保持连接时间过短，导致TCP断开
   而数据库端依旧保持连接等待idle超时；每5分钟查询峰值到达APP时，连接池检查连接不可用，丢弃后重新创建连接；
   但数据库idle远大于5分钟故不释放session，如此往复创建连接导致数据库端连接用完，印证了现象3；

4、回顾升级前，同样是APP->LVS->DB架构，却没有发生异常；猜测是所用的连接池差异导致该事故，以前用的是C3P0，
   升级以后使用HikariCP；经过几轮测试发现C3P0具有连接保活机制，在连接idle是主动发送探测请求到数据库，
   这个探测间隔是60秒，与LVS保持连接时间一致。HikariCP没有idle保活机制，它只在从池中获取连接时检查连接可用性。

5、解决方案：1、调整lvs超时时间，我们的lvs有多个业务公用，且lvs无法单独设置某个VIP的超时时间
             2、直连数据库，绕过lvs超时机制 (仍然要考虑TCP层超时)
             3、调整数据库idle时间，我们公用数据库会影响其他业务
             4、应用层主动闲时探测保活，需要连接池支持：c3p0、druid
             5、加强监控，app探测，日志监控，数据库监控，从多角度早发现问题

6、扩展研究，支持idle保活的连接池有C3P0，阿里druid
c3p0测试通过：网络流传性能比较差，但本场景下表现稳定
```
    <bean id="logDataSource"
          class="com.mchange.v2.c3p0.ComboPooledDataSource"
          destroy-method="close">
        <property name="jdbcUrl" value="${url}" />
        <property name="driverClass" value="${driverClassName}" />
        <property name="user" value="${user}" />
        <property name="password" value="${passwd}" />
        <property name="maxPoolSize" value="${maxPoolSize}" />
        <property name="minPoolSize" value="#{${maxPoolSize}/3}" />
        <property name="acquireIncrement" value="#{${maxPoolSize}/2}" />
        <property name="testConnectionOnCheckin" value="true" />
        <property name="checkoutTimeout" value="120000" />
        <property name="idleConnectionTestPeriod" value="120" /><!--定时检查空闲连接活性-->
        <property name="preferredTestQuery" value="select 1 from dual" /><!--活性探测SQL-->
    </bean>
```
druid测试失败：使用timeBetweenEvictionRunsMillis、minEvictableIdleTimeMillis两个个参数来控制连接的快速逐出，
                   使用minIdle、keepAlive来做idle连接保活，
                   测试发现逐出执行时间并不是设置的时间，测试值大概是设置值的3倍左右；
                   测试发现个别连接既没有执行逐出也没有执行保活动作，直到数据库端session超时，
                   导致一段时间后从数据库端发现连接数小于minIdle，并且在逐渐减少,
                   而从druid监控来看依然有minIdle个连接，并且KACheckCnt正常增加;
```
    <bean id="dataSource" class="com.alibaba.druid.pool.DruidDataSource"
          init-method="init" destroy-method="close">
        <property name="url" value="${url}" />
        <property name="username" value="${user}" />
        <property name="password" value="${passwd}" />
        <property name="maxActive" value="${maxPoolSize}" />
        <property name="initialSize" value="5" />
        <property name="minIdle" value="5" />
        <property name="maxWait" value="60000" />
        <property name="timeBetweenEvictionRunsMillis" value="60000" />
        <property name="minEvictableIdleTimeMillis" value="60000" />
        <property name="validationQuery" value="select 1 from dual" />
        <property name="testWhileIdle" value="true" />
        <property name="testOnBorrow" value="false" />
        <property name="keepAlive" value="true" />
        <property name="filters" value="stat" />
    </bean>
```


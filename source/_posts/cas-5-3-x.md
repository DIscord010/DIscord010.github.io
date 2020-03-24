---
title: CAS 5.3.x 服务端搭建步骤
date: 2019-10-24 13:37:54
tags: [单点登录]
categories: 后端开发
---

### template下载

地址： https://github.com/apereo/cas-overlay-template/tree/5.3 

### keystore配置

CAS服务端默认使用HTTPS，需要配置证书文件。template项目提供的脚本拥有生成证书的命令，直接执行：

```bash
build gencert
```

即会生成证书文件，默认路径为当前盘符目录下的`\etc\cas`中。

### HTTP支持（客户端使用HTTP协议）

修改application.properties配置文件（cas-overlay-template-5.3\src\main\resources），增加：

```Java
cas.tgc.secure=false
cas.serviceRegistry.initFromJson=true
```

修改HTTPSandIMAPS-10000001.json配置文件（cas-overlay-template-5.3\src\main\resources\services）：

```Java
{
  "@class" : "org.apereo.cas.services.RegexRegisteredService",
  "serviceId" : "^(https|imaps|http)://.*",
  "name" : "HTTPS and IMAPS",
  "id" : 10000001,
  "description" : "This service definition authorizes all application urls that support HTTPS and IMAPS protocols.",
  "evaluationOrder" : 10000
}
```

第三行与默认配置相比增加了http协议的支持。

### 项目运行

```bash
build run
```
---
typora-root-url: ../../../../static
typora-copy-images-to: ../../../../static
---

# 概述

在MySQL中，对于client发来的请求，其处理流程分为建链和请求处理两部分，这两个阶段分别称为connection phase和command phase。

MySQL的server-client protocol交互如下：

![MySQL_conn_client-server_protocol](/MySQL_conn_client-server_protocol.png)
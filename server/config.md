# 配置

### 1、 客户端寻址方式
* 代码中指定NameServer地址  
```java
Producer.setNamesrvAddr(“192.168.8.106:9876”);
```
或
```java
Consumer.setNamesrvAddr(“192.168.8.106:9876”);
```

* Java启动参数中指定NameServer地址  
```shell
-Drocketmq.namesrv.addr=192.168.8.106:9876
```

* 环境变量指定NameServer地址·
```shell
export NAMESRV_ADDR=192.168.8.106:9876
```
* http静态服务器寻址  
客户端启动后，会定时访问一个静态的HTTP服务器，地址如下：  
```
http://jmenv.tbsite.net:8080/rocketmq/msaddr
```
这个URL的返回内容如下：
```
192.168.8.106:9876
```
客户端默认每隔2分钟访问一次这个HTTP服务器，并更新本地的NameServer地址。  
URL已经在代码中写死，可通过修改/etc/hosts文件来改变要访问的服务器，例如在/etc/hosts增加如下配置：
```shell
10.232.22.67   jmenv.taobao.net
```
&nbsp;&nbsp;

### 2、 客户端的公共配置类：ClientConfig

|       参数名        |   默认值    |                            说明                            |
|:------------------:|:-----------:|:----------------------------------------------------------:|
| namesrvAddr        |             | NameServer地址列表，多个nameServer地址用分号隔开              |
| clientIP           | 本机IP      | 客户端本机IP地址，某些机器会发生无法识别客户端IP地址情况，需要应用在代码中强制指定 |

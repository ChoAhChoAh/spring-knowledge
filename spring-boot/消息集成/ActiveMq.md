# SpringBoot消息
## ActiveMq
### 1.单机安装（基于ubuntu）
#### 1.1 下载解压
activemq使用java语言实现，因此安装前需要保证环境上有jdk安装。  
本文基于activemq 6.1.2版本（classic）安装，系统为ubuntu20.04。  
activemq的程序包可以在官网查看，根据安装环境下载对应的程序包即可：https://activemq.apache.org/components/classic/download/。
可以先下载后上传linux服务器，也可以获取下载链接通过wget直接下载到服务器。  
下载后可以参考官网文件进行解压：https://activemq.apache.org/components/classic/documentation/getting-started#InstallationProcedureforUnix
可参考如下命令：
```
将程序包解压
tar -zxf apache-activemq-6.1.2-bin.tar.gz
移动到相应目录
mv apache-activemq-6.1.2 /usr/local/activemq
```
#### 1.2 配置
解压后如果需要修改绑定ip和端口，可以进入config目录编辑文件activemq.xml文件
```
进入conf目录
cd /usr/local/activemq/conf
编辑文件
vim activemq.xml
```
在activemq.xml文件中有诸多配置，如需修改绑定ip和端口，查找<transportConnectors>标签，标签包含各种mq协议的链接端口：
```xml
<transportConnectors>
     <!-- DOS protection, limit concurrent connections to 1000 and frame size to 100MB -->
    <transportConnector name="openwire" uri="tcp://10.10.10.10:61616?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="amqp" uri="amqp://10.10.10.10:5672?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="stomp" uri="stomp://10.10.10.10:61613?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="mqtt" uri="mqtt://10.10.10.10:1883?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
    <transportConnector name="ws" uri="ws://10.10.10.10:61614?maximumConnections=1000&amp;wireFormat.maxFrameSize=104857600"/>
</transportConnectors>
```
修改后保存即可。

如果想要activemq管理服务的ip和端口，可以修改config目录下的jetty.xml文件，该文件和activemq.xml在同级目录。
```
vim jetty.xml
```
找到jettyPort关键字的bean标签,修改ip端口后保存即可：
```xml
<bean id="jettyPort" class="org.apache.activemq.web.WebConsolePort" init-method="start">
         <!-- the default port number for the web console -->
    <property name="host" value="10.10.10.10"/>
    <property name="port" value="8161"/>
</bean>

```
如果服务器存在防火墙，请配置端口访问策略，开放端口。

#### 1.3 启动
完成1.2的简单配置后，进入bin目录，运行activemq文件进行启动即可：
```
cd /usr/local/activemq/bin
./activemq start
```
启动后可以通过ps -ef|grep "activemq"查看进程是否存在。

#### 1.4 管理界面
默认管理界面启动端口为8161,如需修改绑定的ip和端口，请查看1.2

### 2.常见问题
#### 2.1 端口冲突
activemq和rabbitmq在amqp协议下的端口都为5672,如果启动后没有activemq的进程，可以通过前台运行模式查看运行报错日志：
```
cd /usr/local/activemq/bin
./activemq console
```
如果服务器有其他应用启动在5672,请更换端口后运行。

### 3.springboot集成使用activemq
#### 3.1 引入依赖
springboot定义了activemq的依赖，引入后可以使用相关接口：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-activemq</artifactId>
</dependency>
```
#### 3.2 配置地址和模式
默认openwire的链接端口为61616, 使用tcp协议链接，如果需要发送，需要指定默认发送地址default-destination, 也可以指定其他地址：
```yaml
spring:
  activemq:
    broker-url: tcp://10.10.10.10:61616
  jms:
    template:
      default-destination: test
```
#### 3.3 代码实现
##### 3.3.1 发送消息
使用JmsMessagingTemplate实现简单的消息发送：
```java
@Service
public class MessageServiceActivemqImpl{

    @Autowired
    private JmsMessagingTemplate messagingTemplate;

    public void sendMessage(String data){
        messagingTemplate.convertAndSend(data);
    }

    public String getMessage(){
        String data = messagingTemplate.receiveAndConvert(String.class);
        return data;
    }
}
```
##### 3.3.2 接收消息
通过 @JmsListener实现监听者，如果需要级联发送，可以使用@SendTo注解。
```java
@Component
public class AmqListener {

    @JmsListener(destination = "test")
    @SendTo("other")
    public String receive(String data){
        return "transfer:"+data;
    }

    @JmsListener(destination = "other")
    public void receiveCascade(String data){
        log.info("cascade data:"+data);
    }

}
```

### 4.高级配置（todo）

### 5.参考
官方文档：https://activemq.apache.org/components/classic/documentation/getting-started#InstallationProcedureforUnix  
linux安装activemq：https://www.cnblogs.com/pinghengxing/p/12593547.html  
端口修改：https://cloud.tencent.com/developer/information/%E4%BD%BF%E7%94%A8%E7%89%B9%E5%AE%9AIP%E5%9C%B0%E5%9D%80%E9%85%8D%E7%BD%AEActiveMQ%205.16.0%20web%E6%8E%A7%E5%88%B6%E5%8F%B0-ask  
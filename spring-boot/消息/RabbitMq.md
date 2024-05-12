# SpringBoot消息
## RabbitMq
### 1.单机安装（基于ubuntu）
#### 1.1 明确版本
rabbitmq基于erlang实现，因此安装rabbitmq前需要安装erlang。  
但安装erlang之前，需要明确rabbitmq和erlang的版本关系，可以参考官网的此说明：https://www.rabbitmq.com/docs/which-erlang，
本文安装的rabbitmq为3.12.14-1版本，因此根据上述的网站查找，依赖的erlang版本为25.0 ~ 26.2.x版本。  

#### 1.2 安装erlang
可以在此网站下载erlang的ubuntu安装包：https://www.erlang-solutions.com/downloads/#  
注意和rabbitmq的对应版本，以及对应系统ubuntu的版本。可以通过lsb_release -a 查看信息，例如ubuntu 20.04.6版本通过前述的命令可以查出如下信息:
```
Distributor ID: Ubuntu
Description:    Ubuntu 20.04.6 LTS
Release:        20.04
Codename:       focal
```
注意Codename要和下载的erlang的系统版本对应，例如：esl-erlang_26.0-1~ubuntu~focal_amd64.deb
下载后，通过dpkg安装，命令如下：
```shell script
root@vm:~# sudo dpkg -i esl-erlang_26.0-1~ubuntu~focal_amd64.deb
```
安装后可以通过whereis erlang进行检查,如果出现安装路径表示安装成功。
```shell script
root@vm:~# whereis erlang
```
```
erlang: /usr/lib/erlang
```
#### 1.3 安装rabbitmq
参考rabbitmq官网的示例步骤进行安装,举例3.12.14-1版本：https://packagecloud.io/rabbitmq/rabbitmq-server/packages/ubuntu/noble/rabbitmq-server_3.12.14-1_all.deb?distro_version_id=284。  
官网给出了不同linux系统的安装命令，ubuntu系统参考上述的官网链接可以使用如下命令：
```shell script
root@vm:~# curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.deb.sh | sudo bash
root@vm:~# sudo apt-get install rabbitmq-server=3.12.14-1
```
安装完成后，建议将本机主机名和ip地址配置
```shell script
root@vm:~# vim /etc/hosts
```
在文件中增加ip和主机名，例如登录时用户@后方的名称为vm（root@vm）,则在hosts文件中增加如下配置并保存。
```
127.0.0.1 vm
```
通过官网提供的方式安装后，rabbitmq的启动等命令会自动加入systemctl，可以通过如下命令进行启动和开机自启动的执行:
```shell script
root@vm:~# systemctl start rabbitmq-server
root@vm:~# systemctl enable rabbitmq-server
```
启动后可以通过进程命令检查rabbitmq是否启动（例如：ps -ef|grep "rabbitmq）

#### 1.4 安装rabbitmq web管理界面
默认情况下rabbitmq不会安装web管理界面，需要手动执行如下命令安装：
```shell script
root@vm:~# rabbitmq-plugins enable rabbitmq_management
```
安装后，通过如下命令重启rabbitmq server以启用管理页面：
```shell script
root@vm:~# systemctl restart rabbitmq-server
```
启动后，可以通过http://localhost:15672访问管理页面，默认使用过quest用户登录（密码也为quest）

#### 1.5 默认端口
默认安装的情况下rabbitmq的amqp服务端口为5672，web管理页面的服务端口为15672。
请配置防火墙策略开启对两个端口的访问。

### 2.springboot集成使用rabbitmq
#### 2.1 引入依赖并增加配置
springboot的父依赖内置了spring-boot-starter-amqp，可以引入该依赖库来实现通过amqp协议接入rabbitmq。  
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-amqp</artifactId>
</dependency>
```

在配置文件中增加rabbitmq的配置：
```yaml
spring:
  rabbitmq:
    port: 5672
    host: 10.10.10.10
    username: xxxx
    password: xxxx
```
#### 2.2 创建queue绑定exchange
##### 2.2.1 direct模式
```java
@Configuration
public class RabbitConfigDirect {
    
    @Bean
    public Queue directQueue(){
        return new Queue("test_queue");
    }
    @Bean
    public DirectExchange directExhange(){
        return new DirectExchange("testExchange");
    }
    @Bean
    public Binding bindDirect(){
        return BindingBuilder.bind(directQueue()).to(directExhange()).with("testDirect");
    }
}
```

##### 2.2.2 topic模式
topic模式支持通配符匹配
```java
@Configuration
public class RabbitConfigTopic {

    @Bean
    public Queue topicQueue(){
        return new Queue("test_topic");
    }
    @Bean
    public TopicExchange topicExhange(){
        return new TopicExchange("testTopicExchange");
    }
    @Bean
    public Binding bindTopic(){
        return BindingBuilder.bind(topicQueue()).to(topicExhange()).with("topic.*.test");
    }
    
}
```

#### 2.3 使用amqp发送消息
直连模式（direct）
```java
@Component
public class MessageSenderRabbitmq{
    
    @Autowired
    private AmqpTemplate amqpTemplate;

    public void sendMessage(String data){
        amqpTemplate.convertAndSend("testExchange","testDirect",data);
    }
    
}
```

topic模式
```java
@Component
public class MessagegSenderRabbitmqTopic {

    @Autowired
    private AmqpTemplate amqpTemplate;
    
    public void sendMessage(String data){
        amqpTemplate.convertAndSend("testTopicExchange","topic.data.test",data);
    }

}
```

#### 2.4 监听消息


### 3.常见问题
#### 3.1 安装erlang时出现依赖问题
如果安装erlang时提示依赖确实，可以通过如下命令更新安装库并安装依赖：
```shell script
sudo apt-get update
sudo apt-get -f -y install
```
如果通过如上命令无法安装erlang的依赖并出现报错，说明下载的erlang安装包版本和系统版本不匹配，重新参考1.2中的说明检查erlang安装包描述的系统版本和实际系统版本的Codename是否一致。

#### 3.2 端口冲突
activemq和rabbitmq在amqp协议下默认使用的端口皆为5672，如果出现rabbitmq启动失败出现如下报错，说明端口冲突：
```
Failed to start Ranch listener {acceptor,{0,0,0,0,0,0,0,0},5672} in ranch_tcp:listen([{port,5672},{ip,{0,0,0,0,0,0,0,0}},inet6,{backlog,128},{nodelay,true},{linger,{true,0}},{exit_on_close,false}]) for reason eacces (permission denied)
```

若在运行代码时发送消息报错协议版本不匹配的错误（如下报错信息），说明可能通过rabbitmq的编码方式向activemq的运行端口发送了消息，检查5672端口是启动的rabbitmq：
```
Exception in thread "pub-sub-pool-2" org.springframework.amqp.AmqpIOException: java.io.IOException
        at org.springframework.amqp.rabbit.connection.RabbitUtils.convertRabbitAccessException(RabbitUtils.java:109)
        at org.springframework.amqp.rabbit.connection.AbstractConnectionFactory.createBareConnection(AbstractConnectionFactory.java:118)
        at org.springframework.amqp.rabbit.connection.CachingConnectionFactory.createConnection(CachingConnectionFactory.java:179)
        at org.springframework.amqp.rabbit.connection.ConnectionFactoryUtils$1.createConnection(ConnectionFactoryUtils.java:77)
        at org.springframework.amqp.rabbit.connection.ConnectionFactoryUtils.doGetTransactionalResourceHolder(ConnectionFactoryUtils.java:121)
        at org.springframework.amqp.rabbit.connection.ConnectionFactoryUtils.getTransactionalResourceHolder(ConnectionFactoryUtils.java:67)
        at org.springframework.amqp.rabbit.connection.RabbitAccessor.getTransactionalResourceHolder(RabbitAccessor.java:100)
        at org.springframework.amqp.rabbit.core.RabbitTemplate.execute(RabbitTemplate.java:402)
        at org.springframework.amqp.rabbit.core.RabbitTemplate.send(RabbitTemplate.java:225)
        at org.springframework.amqp.rabbit.core.RabbitTemplate.convertAndSend(RabbitTemplate.java:242)
        at org.springframework.amqp.rabbit.core.RabbitTemplate.convertAndSend(RabbitTemplate.java:238)
        at com.comp.transports.fix.handlers.Sender$1.run(Sender.java:115)
        at java.util.concurrent.ThreadPoolExecutor$Worker.runTask(ThreadPoolExecutor.java:886)
        at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:908)
        at java.lang.Thread.run(Thread.java:619)
Caused by: java.io.IOException
        at com.rabbitmq.client.impl.AMQChannel.wrap(AMQChannel.java:107)
        at com.rabbitmq.client.impl.AMQConnection.start(AMQConnection.java:261)
        at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:403)
        at com.rabbitmq.client.ConnectionFactory.newConnection(ConnectionFactory.java:423)
        at org.springframework.amqp.rabbit.connection.AbstractConnectionFactory.createBareConnection(AbstractConnectionFactory.java:116)
        ... 13 more
Caused by: com.rabbitmq.client.ShutdownSignalException: connection error; reason: com.rabbitmq.client.MalformedFrameException: AMQP protocol version mismatch; we are version 0-9-1, server sent signature 1,1,0,9
        at com.rabbitmq.utility.ValueOrException.getValue(ValueOrException.java:67)
        at com.rabbitmq.utility.BlockingValueOrException.uninterruptibleGetValue(BlockingValueOrException.java:33)
        at com.rabbitmq.client.impl.AMQChannel$BlockingRpcContinuation.getReply(AMQChannel.java:328)
        at com.rabbitmq.client.impl.AMQConnection.start(AMQConnection.java:246)
        ... 16 more
Caused by: com.rabbitmq.client.MalformedFrameException: AMQP protocol version mismatch; we are version 0-9-1, server sent signature 1,1,0,9
        at com.rabbitmq.client.impl.Frame.protocolVersionMismatch(Frame.java:183)
        at com.rabbitmq.client.impl.Frame.readFrom(Frame.java:120)
        at com.rabbitmq.client.impl.SocketFrameHandler.readFrame(SocketFrameHandler.java:140)
        at com.rabbitmq.client.impl.AMQConnection.readFrame(AMQConnection.java:397)
        at com.rabbitmq.client.impl.AMQConnection$MainLoop.run(AMQConnection.java:425)
```

#### 3.3 使用guest登录失败
官网可以查看默认guest用户不支持非localhost登录，如果需要可以创建新账户：https://www.rabbitmq.com/docs/access-control  
创建账户的命令, linux命令运行：
```
rabbitmqctl add_user test root
rabbitmqctl set_user_tags test administrator
rabbitmqctl set_permissions -p / test ".*" ".*" ".*"
```

#### 3.4 认证失败导致代码客户端链接失败
问题同3.3，guest用户不支持验证，需要创建授权账号链接rabbitmq服务


### 4.配置及其它高阶操作（todo）


### 5.参考
erlang安装：https://blog.csdn.net/laterstage/article/details/131513793?spm=1001.2014.3001.5501  
rabbitmq安装：https://blog.csdn.net/laterstage/article/details/131522924  
5672端口非rabbitmq导致协议不匹配问题：https://groups.google.com/g/rabbitmq-discuss/c/dJEp9q5WnCI  
rabbitmq因为端口冲突启动失败：https://blog.csdn.net/u010402202/article/details/80026501  
rabbitmq的java客户端文档：https://www.rabbitmq.com/tutorials/tutorial-one-java  
rabbitmq链接账户失败：https://blog.csdn.net/leisure_life/article/details/78646211  
dpkg安装deb依赖缺失处理：https://www.cnblogs.com/horizonli/p/5179224.html  
rabbitmq创建账户： https://blog.csdn.net/Mr_Z2017512/article/details/136631454  

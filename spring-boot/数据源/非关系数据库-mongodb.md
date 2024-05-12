# SpringBoot数据源
## 非关系数据库-mongodb
### 1.单体安装（基于ubuntu20.04）
#### 1.1 下载安装包
在官方网站下载安装包上传服务器，注意选择版本和系统类型：https://www.mongodb.com/try/download/community  
也可以通过wget命令在服务器直接下载。  
下载后，在服务器解压程序包，并建议移动到统一的程序安装目录：
```
tar -xvf mongodb-linux-x86_64-ubuntu2004-7.0.9.tgz
mv mongodb-linux-x86_64-ubuntu2004-7.0.9 /usr/local/mongodb7.0.9
```

#### 1.2 创建数据和日志目录
进入安装目录，创建数据和日志目录,并赋予用户目录所有者权限
```
cd /usr/local/mongodb7.0.9/
mkdir -p /usr/local/mongodb7.0.9/data/db
mkdir -p /usr/local/mongodb7.0.9/logs/

chown root /usr/local/mongodb7.0.9/data/db
chown root /usr/local/mongodb7.0.9/logs
```

#### 1.3 添加path
在当前登录用户根路径下创建bash文件，增加mongo的bin目录：
```
vim bash_profile
```
填入如下内容并保存：
```shell script
if [ -f ~/.bashrc ]; then
        . ~/.bashrc
fi
PATH=$PATH:$HOME/bin:/usr/local/mongodb7.0.9/bin
```

#### 1.4 修改配置文件
创建配置文件，可以直接在bin目录下简单创建:
```
cd /usr/local/mongodb7.0.9/bin
vim mongodb.conf
```
可以参考填入如下内容,包含端口、服务地址、数据目录、日志目录、安全验证开关等：
```
port=27017
dbpath=/usr/local/mongodb7.0.9/data/db
logpath=/usr/local/mongodb7.0.9/logs/mongodb.log
fork=true
auth=false
logappend=true
bind_ip=10.0.0.204
```

#### 1.5 启动mongodb
可以将mongodb加入启动命令，也可以使用如下命令简单启动：
```
cd /usr/local/mongodb7.0.9/bin
./mongod -f ./mongodb.conf
```

#### 1.6 安装mongodb的sh客户端
在mongo官网下载：https://www.mongodb.com/try/download/shell
注意选择系统和版本，本文基于ubuntu安装，使用如下命令：
```
sudo dpkg -i mongodb-mongosh_2.2.5_amd64.deb
```
安装后，可以使用如下命令链接到mongodb的服务：
```
mongo 10.0.0.204:27017
```

#### 1.7 创建账户
可以参考：https://blog.csdn.net/kk185800961/article/details/45619863
本文在使用阶段暂时关闭了账户校验。

#### 1.8 可视化客户端使用（Studio 3T）
robo 3t目前已经不在维护，最新的mongodb推荐使用studio 3t可视化操作工具，可以进入官网下载安装：
https://robomongo.org/
##### 1.8.1 创建链接
在studio 3t中选择connect,创建并下一步，输入ip和端口，如果开启了验证，需要在authentication选择鉴权模式，并输入用户名和密码等。  

##### 1.8.2 开启连接后创建集合和操作
开启链接后，可以在collections中创建集合和插入数据
基础的命令如下：
```
db.config.insertMany([{"name":"ooo","type":"1"},{"name":"oo1","type":"2"}]);
db.getCollection("config").find();
db.config.find({type:"1"});
db.config.remove({type:"1"});
db.config.update({type:"1"},{$set:{name:"oodc"}});
db.config.updateMany({type:"1"},{$set:{name:"oodc"}});
```

### 2.spring-boot集成
#### 2.1 引入依赖
springboot内置了mongodb的集成，引入依赖即可：
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-mongodb</artifactId>
</dependency>
```
#### 2.2 配置mongodb
在application.yml中配置mongodb链接等：
```yaml
spring:
  data:
    mongodb:
      uri: mongodb://10.10.10.10/liecho
```

#### 2.3 使用相关接口编程
```java
@SpringBootTest
@ContextConfiguration(classes = {XXXApplication.class} )
public class XXXApplicationTest {
    
    @Autowired
    private MongoTemplate mongoTemplate;

    @Test
    void test_mongo(){
        Config config = new Config();
        config.setName("lql");
        config.setType("233");
        mongoTemplate.save(config);

        ExecutableFindOperation.ExecutableFind<Config> query = mongoTemplate.query(Config.class);
        List<Config> all = query.all();
        System.out.println(all);
    }

}
```

### 3.常见问题
暂无

### 4.高级配置和运维(todo)


### 5.参考
linux安装mongodb: https://blog.csdn.net/a342874650/article/details/133859284
                 https://www.cnblogs.com/hmwh/p/13795262.html
mongodb安全： https://blog.csdn.net/kk185800961/article/details/45619863
             https://blog.csdn.net/Innocence_0/article/details/136375030
studio 3t官网：https://robomongo.org/
sutdio 3t使用：https://blog.csdn.net/weixin_39999535/article/details/81383196


# spring集成log4j2+slf4j+lombok
## slf4j 1.7.36 + log4j 2.17.2 + lombok 1.18.32
step1: 引入依赖,slf4j需要引入api，log4j2需要引入log4j2-core、log4j2-slf4j-impl、log4j-api：
```xml
<dependencies>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.32</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.slf4j</groupId>
        <artifactId>slf4j-api</artifactId>
        <version>1.7.36</version>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-core</artifactId>
        <version>2.17.2</version>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-slf4j-impl</artifactId>
        <version>2.17.2</version>
    </dependency>
    <dependency>
        <groupId>org.apache.logging.log4j</groupId>
        <artifactId>log4j-api</artifactId>
        <version>2.17.2</version>
    </dependency>
</dependencies>
```
step2: 在资源目录src/main/resources下编写log4j2.xml,最基本的控制台打印如下：
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Configuration status="INFO">
    <Appenders>
        <Console name="Console" target="SYSTEM_OUT">
            <PatternLayout pattern="%d{yyyy-MM-dd HH:mm:ss} %-5p %c{1}:%L - %m%n"/>
        </Console>
    </Appenders>
    <Loggers>
        <Root level="info">
            <AppenderRef ref="Console"/>
        </Root>
    </Loggers>
</Configuration>
```
step3: 使用lombok的@Slf4j注解打印日志：
```java
@Slf4j
public class CustomeAdvice {
    public void preInvoke(){
        log.info("pre invoke");
    }
    public void postInvoke(){
        log.info("post invoke");
    }
}
```
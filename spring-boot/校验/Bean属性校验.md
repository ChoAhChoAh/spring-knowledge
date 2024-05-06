# SpringBoot校验
## Bean属性校验
### 1.JSR303校验
引入jdk的校验api和hibernate的校验实现：
```xml
<dependencies>
    ...
    <dependency>
         <groupId>javax.validation</groupId>
         <artifactId>validation-api</artifactId>
    </dependency>
    <dependency>
        <groupId>org.hibernate.validator</groupId>
        <artifactId>hibernate-validator</artifactId>
    </dependency>
    ...
</dependencies>
```

使用Spring提供的@Validated注解+jdk提供的校验注解（在javax.validation.constraints包下）,举例如下：
```java
@Component
@ConfigurationProperties(prefix = "xxx")
@Data
@Validated
public class XXXConfig {
    
    @Max(9999)
    @Min(7777)
    private int range;

}
```
如果使用上述代码运行，如果给range设置小于7777或者大于9999的情况下，容器启动会报错。
```
Binding to target org.springframework.boot.context.properties.bind.BindException: Failed to bind properties under 'range' to xxx.xxx.xxx.xxxConfig failed:

    Property: xxx.range
    Value: "10000"
    Origin: class path resource [application.yml] - 4:9
    Reason: 最大不能超过9999
```
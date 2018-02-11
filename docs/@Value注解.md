### 用法
外部参数对应的property,被spring加载的properties文件
```java
@Value("${key}")
```
SpEL表达式对应的内容
```java
@Value("#{config['key']}")
```
对于SpEL方式的变量，需要在xml加入下列方式配置
```xml
<util:properties id="config" location="classpath*:config.properties"></util:properties>
```
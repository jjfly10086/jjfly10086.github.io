### -D参数
-D代表（Properties属性）

使用命令行设置属性-D的正确方法是：

mvn -DpropertyName=propertyValue clean/package
如果propertyName不存在pom.xml，它将被设置。
如果propertyName已经存在pom.xml，其值将被作为参数传递的值覆盖-D。
要发送多个变量，请使用多个空格分隔符加-D：
如果你的pom.xml如下：
```xml
<properties>
        <theme>myDefaultTheme</theme>
</properties>
```
    
那么在这个执行过程中mvn -Dtheme=halloween clean package会覆盖theme的值，具有如下效果：
```xml
<properties>
        <theme>halloween</theme>
</properties>
```
类似于java虚拟机参数Java System Property中的-D参数

### -P参数
也就是说在<profiles>指定的<id>中，可以通过-P进行传递或者赋值。
修改profile中参数
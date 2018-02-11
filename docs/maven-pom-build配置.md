### assembly打包 jar包配置
参考链接：http://blog.csdn.net/xiaojiesu/article/details/51871705

### 1.pom.xml中build节点配置
```xml
<project>
    <properties>
        <config_env>dev</config_env>
    </properties>
    <!--打包插件-->
     <build>
             <!--过滤指定配置文件，可在maven package时指定-D参数修改pom中默认的config_env参数-->
             <filters>
                 <filter>src/main/filters/filter.${config_env}.properties</filter>
             </filters>
             <resources>
                 <resource>
                     <!--配置文件源目录-->
                     <directory>src/main/resources</directory>
                     <!--配置文件打包目录-->
                     <targetPath>${project.build.directory}/classes</targetPath>
                     <!--在resources目录下的配置文件中的替代标签是否使用filters中过滤的配置-->
                     <filtering>true</filtering>
                 </resource>
             </resources>
             <plugins>
                 <!--resources plugin-->
                 <plugin>
                     <groupId>org.apache.maven.plugins</groupId>
                     <artifactId>maven-resources-plugin</artifactId>
                     <version>2.6</version>
                     <configuration>
                         <!--设置过滤的配置文件的编码方式-->
                         <encoding>UTF-8</encoding>
                     </configuration>
                 </plugin>
                 <!--compiler plugin-->
                 <plugin>
                     <groupId>org.apache.maven.plugins</groupId>
                     <artifactId>maven-compiler-plugin</artifactId>
                     <version>3.1</version>
                     <configuration>
                         <source>1.8</source>
                         <target>1.8</target>
                         <!-- 设置源文件编码方式 -->
                         <encoding>UTF-8</encoding>
                     </configuration>
                 </plugin>
                 <!-- The configuration of maven-jar-plugin -->
                 <plugin>
                     <groupId>org.apache.maven.plugins</groupId>
                     <artifactId>maven-jar-plugin</artifactId>
                     <version>2.4</version>
                     <!-- The configuration of the plugin -->
                     <configuration>
                         <!-- Configuration of the archiver -->
                         <archive>
                             <!--生成的jar中，不要包含pom.xml和pom.properties这两个文件-->
                             <addMavenDescriptor>false</addMavenDescriptor>
                             <!-- Manifest specific configuration -->
                             <manifest>
                                 <!--是否要把第三方jar放到manifest的classpath中-->
                                 <addClasspath>true</addClasspath>
                                 <!-- 生成的manifest中classpath的前缀，因为要把第三方jar放到lib目录下，所以classpath的前缀是lib/-->
                                 <classpathPrefix>lib/</classpathPrefix>
                                 <!--应用的main class-->
                                 <mainClass>com.jwh.demo.StartServer</mainClass>
                             </manifest>
                         </archive>
                         <!--需要打包进jar的目录-->
                         <includes>
                             <include>com/**</include>
                             <include>resources/**</include>
                         </includes>
                     </configuration>
                 </plugin>
     
                 <!-- The configuration of maven-assembly-plugin -->
                 <plugin>
                     <groupId>org.apache.maven.plugins</groupId>
                     <artifactId>maven-assembly-plugin</artifactId>
                     <version>2.4</version>
                     <!-- The configuration of the plugin -->
                     <configuration>
                         <!-- Specifies the configuration file of the assembly plugin -->
                         <descriptors>
                             <descriptor>src/main/assembly/assembly.xml</descriptor>
                         </descriptors>
                     </configuration>
                     <executions>
                         <execution>
                             <id>make-assembly</id>
                             <phase>package</phase>
                             <goals>
                                 <goal>single</goal>
                             </goals>
                         </execution>
                     </executions>
                 </plugin>
             </plugins>
     </build>
</project>
```
 ### 2.assembly.xml配置
 ```xml
 <!--assembly.xml配置-->
 <assembly>
     <id>bin</id>
     <!--打包目录下是否再包一层根目录-->
     <includeBaseDirectory>false</includeBaseDirectory>
     <!-- 最终打包成一个用于发布的tar.gz文件 -->
     <formats>
         <format>tar.gz</format>
     </formats>
     <!--添加依赖到tar.gz压缩包的/lib目录下-->
     <!-- Adds dependencies to zip package under lib directory -->
     <dependencySets>
         <dependencySet>
             <!--不使用项目的artifact(即不将当前项目的jar打包到/lib下)，第三方jar不要解压，打包进tar.gz文件的lib目录-->
             <useProjectArtifact>false</useProjectArtifact>
             <outputDirectory>/lib</outputDirectory>
             <unpack>false</unpack>
         </dependencySet>
     </dependencySets>
     <fileSets>
         <!-- 把项目相关的说明文件，打包进zip文件的根目录 -->
         <!--<fileSet>-->
         <!--<directory>${project.basedir}</directory>-->
         <!--<outputDirectory>/</outputDirectory>-->
         <!--</fileSet>-->
          <!--项目启动文件-->
          <fileSet>
              <directory>${project.basedir}/src/main/bin</directory>
          <outputDirectory>/bin</outputDirectory>
         </fileSet>
         <!-- 把项目的配置文件，打包进tar.gz文件的config目录 -->
         <fileSet>
             <directory>${project.build.directory}/classes/</directory>
             <outputDirectory>/conf</outputDirectory>
             <includes>
                 <include>*.xml</include>
                 <include>*.properties</include>
             </includes>
         </fileSet>
         <!-- 把项目自己编译出来的jar文件，打包进tar.gz文件的根目录 -->
         <fileSet>
             <directory>${project.build.directory}</directory>
             <outputDirectory>/</outputDirectory>
             <includes>
                 <include>*.jar</include>
             </includes>
         </fileSet>
     </fileSets>
 </assembly>
```
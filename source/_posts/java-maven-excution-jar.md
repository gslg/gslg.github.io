---
title: java使用maven打包为可执行的jar包
author: gslg
date: 2019-07-08 16:30:22
tags:
---
一般使用`maven-assembly-plugin`插件即可
<!--more-->
{% codeblock pom.xml %}
    ....
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>2.4</version>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                    <archive>
                        <manifest>
                           <!--主类入口--> 
                          <mainClass>com.example.ExcelExport</mainClass>
                        </manifest>
                    </archive>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source> <!-- 源代码使用的JDK版本 -->
                    <target>1.8</target> <!-- 需要生成的目标class文件的编译版本 -->
                    <encoding>UTF-8</encoding><!-- 字符集编码 -->
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
{% endcodeblock %}

这样，当我们使用`mvn clean package`打包时，会在target目录下生成两个jar包:
`**.jar`和`***-jar-with-dependencies.jar`，前者是不含依赖的包，不能直接运行,后者是可直接执行的jar包,也可以直接使用`mvn clean compile assembly:single`命令只生成可执行的jar包.

---
title: 【Java 基础】使用命令编译简单的 Java 项目
date: 2019-08-18
draft: false
---

## CLASSPATH

classpath 可以理解为 java 程序在编译或运行时需要用到的其他 class 文件或 jar 包及配置文件的路径。

JDK1.5 之后就不用再配置者个环境变量，JRE 会自动搜索当前路径下的 class 文件，自动加载 dt.jar 和 tools.jar。

如果手动地在命令行下进行编译和运行时，根据项目结构，可以使用 -classpath 指定具体的目录。

## 项目结构

![source structure](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2019-08/2019-08-18_source.png)

* src：存放源码，分为两个包：hello 和 hi，Test 属于 hi 包，作为启动入口
* target：编译后 class 文件的位置
* lib：Test 中引入的第三方 jar 包
* resource：第三方 jar 包需要的配置文件

Hello.java

```java
package hello;

public class Hello{

    public void hello(){
        System.out.println("hello");
    }
}
```

Hi.java

```java
package hi;

public class Hi{

    public void hi(){
        System.out.println("hi");
    }
}
```

Test.java

```java
package hi;

import hello.*;
import org.slf4j.*;

public class Test{

    private static Logger logger = LoggerFactory.getLogger(Test.class);

    public static void main(String[] args){

        Hello hello = new Hello();
        Hi hi = new Hi();
        hello.hello();
        hi.hi();
        logger.info("This is slf4j");
    }
}
```

logback.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>

    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <root level="INFO">
        <appender-ref ref="CONSOLE" />
    </root>

</configuration>
```

```shell
```

## 编译

Test.java 依赖 Hello.java、Hi.java 和 lib 中的 jar 包，先编译 Hello.java 和 Hi.java。

**需要注意的是，编译和运行时的所有命令都是在当前目录下执行，也就是和 resource、lib 以及 hello 包、 hi 包同级**。

```shell
javac src/hello/hello.java -d target

javac src/hi/hi.java -d target
```

-d：指定编译后 class 文件的位置，这里指定为预先建立的 target 目录。

```shell
javac -cp target:lib/logback-core-1.2.3.jar:lib/logback-classic-1.2.3.jar:lib/slf4j-api-1.7.26.jar src/hi/Test.java -d target
```

-cp：指定了编译 Test.java 时需要的其他类的路径，这里指定了 target 和 lib 目录，多个路径使用:分隔，jar 包需要逐一指出。

```shell
javac -cp lib/logback-core-1.2.3.jar:lib/logback-classic-1.2.3.jar:lib/slf4j-api-1.7.26.jar src/hello/Hello.java src/hi/Hi.java src/hi/Test.java -d target
```

如果同时编译三个源文件，则不需要在 classpath 中添加 target 目录，只需要指定 Test.java 依赖的第三方 jar 包即可。三个源文件的顺序没有规定。

编译后的项目结构如下图：

![compiled structure](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2019-08/2019-08-18_compile.png)

## 运行

运行 Test.class：

```shell
java -cp resource:target:lib/logback-classic-1.2.3.jar:lib/logback-core-1.2.3.jar:lib/slf4j-api-1.7.26.jar hi.Test
```

运行时不仅需要指定 Test.class 所在的目录 target，还需要指定依赖的第三方 jar 包和其配置文件的位置。

由于 target 目录也包含了 Hello.class 和 Hi.class，所以不需要明确指出这两个依赖项。

Test.class 执行时需要添加包名。

运行结果如下图：

![result](https://blog-pankekp-image.oss-cn-beijing.aliyuncs.com/2019-08/2019-08-18_result.png)

[TOC]

# Maven之POM文件

# Maven简介

> Maven，单词源于九世纪的中欧，原意为致力于传道受业解惑的某一领域的专家。

Maven是构建java工程的主流工具，包括Java工程的结构化管理、编译、打包、发布、测试等。

## Maven的特点
* 提供简单的构建入口
* 提供统一的构建体系
* 提供完善的项目生命周期管理
* 提供丰富的扩展插件
* 提供便捷的依赖管理
* 提供工程化管理的最佳实践

## Maven的历史
> Maven 1/2已不再更新，建议使用Maven 3+。
> Maven 3相比Maven 2主要修复了多处bug和多处改进，比如优化的包依赖和插件依赖管理、优化的日志输出、支持并行编译、更为合理和标准的POM结构等。

* 创建于2002年，至今已有3个大版本。最新版本为3.6.0，更新时间为2018-10-24。
* 在Java编译和项目管理的工具发展史中也出现过其他一些身影，比如Ant（编译打包）、Ivy（依赖包管理）
现在较新的Java项目管理工具如Gradle在Maven的基础上更进行了多处改进，比如去除繁杂的xml管理、更灵活的生命周期管理（Maven为固定生命周期流程 + 插件钩子）、多来源方式的依赖包（Maven必须为指定标准的Metadata，无论公仓还是私仓）。
* 选择Maven一是考虑到生态环境的成熟度，此外还有参考其他团队的选型和学习成本的考量。
* 更多Gradle与Maven的对比可以参考[Gradle VS Maven](https://gradle.org/maven-vs-gradle/)。

## Maven VS MSBuild
> MSBuild更像一个单纯的编译工具，虽然可以以自定义Task的方式在编译的各生命周期进行扩展，但是缺少依赖包的管理及自动化测试。在C#中与Maven的概念更为相似的是Maven的迁移项目NMaven。

Maven具有便捷的依赖包管理功能，在Java中Maven的地位相当于Python中的Pip、NodeJS中的npm、C#中的Nuget。
除此之外，Maven也囊括了项目的编译及发布等生命周期各环节的管理，与C#中的MSBuild比较主要有如下异同：

| 功能点     | Maven | MSBuild |
| ---------- | ----- | ------- |
| 编译       | ✅     | ✅       |
| 打包       | ✅     | ✅       |
| 发布       | ✅     | ✅       |
| 事件钩子   | ✅     | ✅       |
| 依赖包管理 | ✅     | ❌       |
| 自动化测试 | ✅     | ❌       |

## Maven生命周期

- **validate**: validate the project is correct and all necessary information is available
- **compile**: compile the source code of the project
- **test**: test the compiled source code using a suitable unit testing framework. These tests should not require the code be packaged or deployed
- **package**: take the compiled code and package it in its distributable format, such as a JAR.
- **integration-test**: process and deploy the package if necessary into an environment where integration tests can be run
- **verify**: run any checks to verify the package is valid and meets quality criteria
- **install**: install the package into the local repository, for use as a dependency in other projects locally
- **deploy**: done in an integration or release environment, copies the final package to the remote repository for sharing with other developers and projects.

# POM文件

> POM = Project Object Model即项目对象模型，是Maven赖以对项目进行管理的依据。

```b
mvn archetype:generate -DgroupId=com.mycompany.app -DartifactId=my-app -DarchetypeArtifactId=maven-archetype-quickstart -DarchetypeVersion=1.4 -DinteractiveMode=false
```
![img](http://172.30.64.71:8092/download/attachments/21587726/image2019-2-21_14-24-24.png?version=1&modificationDate=1550729558537&api=v2)
## POM文件基础配置
```b
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
 
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>my-project</artifactId>
  <version>1.0</version>
</project>
```
* model 4.0.0是目前Maven2&3唯一支持的解析版本，可以视其为固定节点。
## POM文件之dependencies
```b
<dependencies>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.12</version>
      <type>jar</type>
      <scope>test</scope>
      <optional>true</optional>
	  <exclusions>
        <exclusion>
          <groupId>org.apache.maven</groupId>
          <artifactId>maven-core</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    ...
  </dependencies>
```
* 一个完整的Maven Coordinate由{groupdId}{artifactId}{version}\[{classifier}\]\[{type}\]组成
* 依赖的optional配置表示当其他项目引用本项目时，该项为非必须项
* version的依赖可以根据实际情况指定范围，类似nuget中的package.json配置
![](http://172.30.64.71:8092/download/attachments/21587726/image2019-2-21_15-11-32.png?version=1&modificationDate=1550732386326&api=v2)
* exclusion支持通配符*，可以完全摒弃所有深层依赖自行管理
  
## POM文件之properties
```b
  <properties>
    <maven.compiler.source>1.7</maven.compiler.source>
    <maven.compiler.target>1.7</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
  </properties>
```
* 属性实际是一种占位符；使用properties节点定义，使用${property}进行占位替换
* Maven内置了一些默认属性可供直接使用，如${env.PATH}、${project.version}、${java.home}
## POM文件之repositories
```b
  <repositories>
    <repository>
      <releases>
        <enabled>false</enabled>
        <updatePolicy>always</updatePolicy>
        <checksumPolicy>warn</checksumPolicy>
      </releases>
      <snapshots>
        <enabled>true</enabled>
        <updatePolicy>never</updatePolicy>
        <checksumPolicy>fail</checksumPolicy>
      </snapshots>
      <id>codehausSnapshots</id>
      <name>Codehaus Snapshots</name>
      <url>http://snapshots.maven.codehaus.org/maven2</url>
      <layout>default</layout>
    </repository>
  </repositories>
```
* release和snapshot分别对应了两种包类型管理策略，可以分别控制开启或停用
* checksumPolicy即对sha1文件进行校验失败后的处理策略
* updatePolicy即根据lastUpdated文件进行更新的策略，支持每天或指定分钟间隔进行依赖包的更新
## POM文件的继承

```b
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
 
  <groupId>org.codehaus.mojo</groupId>
  <artifactId>my-parent</artifactId>
  <version>2.0</version>
  <packaging>pom</packaging>


  <modules>
    <module>my-project</module>
    <module>another-project</module>
    <module>third-project/pom-example.xml</module>
  </modules>
</project>
```
```b
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>


  <groupId>com.eastmoney.dclogs</groupId>
  <artifactId>my-project</artifactId>

  <parent>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>my-parent</artifactId>
    <version>2.0</version>
    <relativePath>../my-parent</relativePath>
  </parent>
</project>
```
* packaging为pom，表示这个文件为父节点；parent用于指定父节点的Coordinate及路径
* 应将父节点POM文件理解为一个与jar包同等对象，只是这个对象仅用于描述项目信息；Maven项目本身有一个内置POM对象，包含一些默认值，具体可参考[The Super POM](http://maven.apache.org/pom.html)
* 使用modules可以让Maven自行管理各modules之间的依赖关系；modules是用于管理模块，parent是用于管理继承，二者之间没有必然的联系

# 参考资源

* [pom文件cookbook](https://maven.apache.org/ref/3.6.0/maven-model/maven.html)
* [pom文件详解](https://maven.apache.org/pom.html) 
  
# FAQ
> 我该使用Repository还是distributionManager/Repository?

需要下载依赖包时，指定Maven仓库地址，使用Repository；需要上传公共库时，指定Maven仓库地址，使用distributionManager/Repository

----

> 我的项目中指定了依赖包如下，在编译时查询该依赖包的顺序是怎么样的?
```b
	<dependency>
      <groupId>com.mycompany</groupId>
      <artifactId>myapp</artifactId>
      <version>1.0</version>
    </dependency>
```

本地repository => 项目repository => Maven-Setting.xml的repository

----

> 暂时无法使用仓库进行下载时，如何手动添加我的依赖包?

* a. 手动添加目录及POM文件
* b. 使用dependency的scope=system + systemPath
* c. 使用如下Maven命令进行手动安装
```b
mvn install:install-file -Dfile=non-maven-proj.jar -DgroupId=some.group -DartifactId=non-maven-proj -Dversion=1 -Dpackaging=jar
```

----

> 我的项目依赖包含A与B，依赖包A与B又依赖了logback日志模块，但是版本冲突导致编译失败，如何解决?

使用exclusions节点进行排除

----


> dependency与dependencyManagement的区别和使用场景?

dependency用于管理当前项目依赖包且会被后代POM文件直接继承；dependencyManagement是用于管理子module的依赖包，后代POM文件使用时必须至少指定对应groupId与artifactId
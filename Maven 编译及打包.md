# Maven 编译及打包

## 预备知识点

### Build Lifecycle、Phase与Goal

- ***Build Lifecycle***即Maven中项目管理的生命周期，Maven内置了三种生命周期：`default，clean，site`。
- ***Phase***即生命周期中的各个阶段；以`default`为例，其生命周期分为```validate，compile，test，package，verify，install，deploy```等几个阶段，如package即表示生命周期中的打包阶段。
- ***Goal***即各个Phase所定义的执行步骤；这些具体的执行步骤通常由Maven插件提供，使用时的命令格式通常为```plugin:goal```。

> 综上所述，一个完整的生命周期***Build Lifecycle***由一个或多个阶段***Phase***组成，一个阶段***Phase***由零个或多个步骤***Goal***组成；同一个步骤***Goal***可以从属于零个或多个阶段***Phase***；一个插件可以同时提供多个***Goal***，可以将其理解为一个插件根据不同参数以提供不同的能力。

### BaseBuild、ProjectBuild、ProfileBuild

* ***ProjectBuild***即为POM文件中的build节点，对应Maven项目的构建配置管理
* ***ProfileBuild***即针对不同环境设置的项目构建配置管
* ***BaseBuild***即build节点与profiles/profile/build节点的一系列共有属性

## 配置说明

> BaseBuild即POM文件中build节点与profiles/profile/build节点的一系列共有属性

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      https://maven.apache.org/xsd/maven-4.0.0.xsd">
  ...
  <!-- "Project Build" contains more elements than just the BaseBuild set -->
  <build>...</build>
 
  <profiles>
    <profile>
      <!-- "Profile Build" contains a subset of "Project Build"s elements -->
      <build>...</build>
    </profile>
  </profiles>
</project>
```

### 基础属性

```
<build>
  <defaultGoal>install</defaultGoal>
  <directory>${basedir}/target</directory>
  <finalName>${artifactId}-${version}</finalName>
  <filters>
    <filter>filters/filter1.properties</filter>
  </filters>
  ...
</build>
```

* ***defaultGoal***，默认构建目标；这里的目标可以是```goal```（构建指令如```jar```），也可以是```phase```（生命周期阶段如```install```）。
* ***directory***，输出文件目录；默认为`${basedir}/target`，若使用相对路径，则当前目录为POM文件所在目录。
* ***finalName***，输出文件名；注意此处并不代表最终的完整的文件名，其也依赖于插件的执行行为；
* ***filters***，环境变量列表；其指定的properties文件按行定义环境变量，如`name=value`，对应的环境变量只能在`build/resources`节点中使用；在Maven中默认的`filter`文件路径为`${basedir}/src/main/filters/`

### 资源文件

```
<build>
    ...
    <resources>
      <resource>
        <targetPath>${project.build.outputDirectory}/conf</targetPath>
        <filtering>false</filtering>
        <directory>${basedir}/src/main/resources/conf</directory>
        <includes>
          <include>configuration.xml</include>
        </includes>
        <excludes>
          <exclude>**/*.properties</exclude>
        </excludes>
      </resource>
    </resources>
    <testResources>
      ...
    </testResources>
    ...
</build>
```

* ***targetPath***，当前资源文件最终输出目录。
* ***filtering***，表示当前资源文件配置节点下，是否启用`build/filters`下的环境变量。
* ***directory***，资源文件目录，可使用相对路径。
* ***includes***，在当前`directory`下需要包含的文件列表。
* ***excludes***，在当前`directory`下需要排除的文件列表。
* ***testResources***，配置项同***resources***，只不过是用于生命周期中的test阶段。

### 插件管理

```
<build>
    ...
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>2.6</version>
        <extensions>false</extensions>
        <inherited>true</inherited>
        <configuration>
          <items combine.children="append">
            <!-- combine.children="merge" is the default -->
            <item>child-1</item>
          </items>
          <properties combine.self="override">
            <!-- combine.self="merge" is the default -->
            <childKey>child</childKey>
          </properties>
        </configuration>
        <dependencies>...</dependencies>
        <executions>
          <execution>
            <id>echodir</id>
            <goals>
              <goal>run</goal>
            </goals>
            <phase>verify</phase>
            <inherited>false</inherited>
            <configuration>
              <tasks>
                <echo>Build Dir: ${project.build.directory}</echo>
              </tasks>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
    
    <pluginManagement>
      <plugins>
        <plugin>
          <groupId>org.apache.maven.plugins</groupId>
          <artifactId>maven-jar-plugin</artifactId>
          <version>2.6</version>
          <executions>
            <execution>
              <id>pre-process-classes</id>
              <phase>compile</phase>
              <goals>
                <goal>jar</goal>
              </goals>
              <configuration>
                <classifier>pre-process</classifier>
              </configuration>
            </execution>
          </executions>
        </plugin>
      </plugins>
    </pluginManagement>
    ...
</build>
```

* ***groupId，artifactId，version***，插件坐标项
* extensions
* inherited，作为父级POM文件是否向下传递该插件配置
* configuration，插件自身的配置属性，依插件的不同而不同

> configuration的继承规则有三种：merge，append，override，默认为merge
>
> 继承规则可以针对每项configuration进行独立设置
>
> configuration的merge继承规则为：异项合并，同项覆盖

* dependencies，插件的依赖包，可以针对一些特殊情况进行针对性的配置，如依赖包版本或者exclusions
* executions，针对不同goal进行的独立配置集

#### Maven-Resources-Plugin

> 该插件支持三个goal命令：resources:resources、resources:testResources、resources:copy-resources
>
> 分别用于复制main\resources、test\resources以及自定义资源目录

#### Maven-Jar-Plugin

> 该插件主要用于生命周期的package阶段，其配置项基于Apache Maven Archiver，具体可参考资源链接。

```
<archive>
  <addMavenDescriptor/>
  <compress/>
  <forced/>
  <index/>
  <manifest>
    <addClasspath/>
    <addDefaultEntries/>
    <addDefaultImplementationEntries/>
    <addDefaultSpecificationEntries/>
    <addBuildEnvironmentEntries/>
    <addExtensions/>
    <classpathLayoutType/>
    <classpathPrefix/>
    <customClasspathLayout/>
    <mainClass/>
    <packageName/>
    <useUniqueVersions/>
  </manifest>
  <manifestEntries>
    <key>value</key>
  </manifestEntries>
  <manifestFile/>
  <manifestSections>
    <manifestSection>
      <name/>
      <manifestEntries>
        <key>value</key>
      </manifestEntries>
    <manifestSection/>
  </manifestSections>
  <pomPropertiesFile/>
</archive>
```

* mainfest\addClasspath，是否添加classpath路径，默认false
* mainfest\classpathPrefix，在所有的classpath中添加指定前缀
* mainfest\mainClass，程序入口即main函数所在类
* manifestEntries，需要添加在classpath中的值，否则会提示引用异常或查找不到配置文件

#### Maven Assembly Plugin

> Assembly Plugin用于将编译后的包进行打包组装，便于发行使用，其配置项主要基于assembly descriptor，具体可参考资源链接。

```
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-assembly-plugin</artifactId>
	<version>2.2-beta-5</version>
	<!-- The configuration of the plugin -->
	<configuration>
		<finalName>${project.artifactId}-${project.version}-${maven.build.timestamp}</finalName>
		<!-- Specifies the configuration file of the assembly plugin -->
		<descriptors>
			<descriptor>src/main/resources/package.xml</descriptor>
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
```

```
<assembly>
    <id>bin</id>
    <formats>
        <format>zip</format>
    </formats>
    <dependencySets>
        <dependencySet>
            <!--
               不使用项目的artifact，第三方jar不要解压，打包进zip文件的lib目录
           -->
            <outputDirectory>lib</outputDirectory>
            <unpack>false</unpack>
        </dependencySet>
    </dependencySets>
    <fileSets>
        <!-- 把项目相关的说明文件，打包进zip文件的根目录 -->
        <fileSet>
            <directory>${project.basedir}</directory>
            <outputDirectory>/</outputDirectory>
            <includes>
                <include>README*</include>
                <include>LICENSE*</include>
                <include>NOTICE*</include>
            </includes>
        </fileSet>

        <!-- 把项目的配置文件，打包进zip文件的config目录 -->
        <fileSet>
            <directory>src/main/resources/conf</directory>
            <outputDirectory>conf</outputDirectory>
            <filtered>true</filtered>
            <includes>
                <include>server.properties</include>
                <include>server-router.json</include>
                <include>log4j.properties</include>
            </includes>
        </fileSet>

        <!-- 把项目的脚本文件目录（ src/main/scripts ）中的启动脚本文件，打包进zip文件的跟目录 -->
        <fileSet>
            <directory>${project.build.scriptSourceDirectory}</directory>
            <outputDirectory></outputDirectory>
            <includes>
                <include>startup.*</include>
            </includes>
        </fileSet>

        <!-- 把项目自己编译出来的jar文件，打包进zip文件的根目录 -->
        <fileSet>
            <directory>${project.build.directory}</directory>
            <outputDirectory></outputDirectory>
            <includes>
                <include>*.jar</include>
            </includes>
        </fileSet>
    </fileSets>
</assembly>
```

* formats

```
"zip" - Creates a ZIP file format
"tar" - Creates a TAR format
"tar.gz" or "tgz" - Creates a gzip'd TAR format
"tar.bz2" or "tbz2" - Creates a bzip'd TAR format
"tar.snappy" - Creates a snappy'd TAR format
"tar.xz" or "txz" - Creates a xz'd TAR format
"jar" - Creates a JAR format
"dir" - Creates an exploded directory format
"war" - Creates a WAR format
```

* dependencySets\dependencySet\outputDirectory
* dependencySets\dependencySet\unpack，仅支持解压 jar, zip, tar.gz, tar.bz
* fileSets\fileSet\directory，设置当前配置对应搜索目录
* fileSets\fileSet\outputDirectory，设置当前配置对应输出目录
* fileSets\fileSet\includes，需要包含的文件，支持通配符
* fileSets\fileSet\excludes，需要排除的文件，支持通配符，同匹配条件excludes具有更高的优先级

### 路径配置

```
<build>
    ...
    <sourceDirectory>${basedir}/src/main/java</sourceDirectory>
    <scriptSourceDirectory>${basedir}/src/main/scripts</scriptSourceDirectory>
    <testSourceDirectory>${basedir}/src/test/java</testSourceDirectory>
    <outputDirectory>${basedir}/target/classes</outputDirectory>
    <testOutputDirectory>${basedir}/target/test-classes</testOutputDirectory>
    ...
</build>
```

> 路径设置仅从属于Project Build节点，在ProfileBuild节点中无法使用

### 扩展配置

```
<extensions>
    <extension>
        <groupId>org.apache.maven.wagon</groupId>
        <artifactId>wagon-ftp</artifactId>
        <version>1.0-alpha-3</version>
    </extension>
</extensions>
```

> 扩展配置指定的依赖包将在build过程中被加载

## 参考资源

[Maven项目管理生命周期](http://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html)

[Maven内置生命周期](http://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#Lifecycle_Reference)

[Packaging对应的default生命周期插件列表](https://maven.apache.org/ref/3.6.0/maven-core/default-bindings.html)

[Maven插件开发手册](http://maven.apache.org/plugin-developers/index.html)

[Maven插件列表](http://maven.apache.org/plugins/index.html)

[Maven-Resources-Plugin](http://maven.apache.org/plugins/maven-resources-plugin/)

[Maven-Jar-Plugin](http://maven.apache.org/plugins/maven-jar-plugin/)

[Maven-Jar-Plugin Configuration](http://maven.apache.org/shared/maven-archiver/)

## FAQ

> 分析以下Maven命令的执行生命周期

```shell
mvn clean dependency:copy-dependencies package
```

答：执行名为`clean`的`lifecycle` -> 执行插件`dependency`的`goal`，`copy-dependencies` -> 执行名为`package`的`phase`

----

> 解释在如下配置中二个complie分别代表什么含义

```
<build>
        <defaultGoal>compile</defaultGoal>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-resources-plugin</artifactId>
                <version>2.7</version>
                <configuration>
                    <encoding>${project.build.sourceEncoding}</encoding>
                </configuration>
                <executions>
                    <execution>
                        <phase>compile</phase>
                        
                    </execution>
                </executions>
            </plugin>
        </plugins>
<build>
```

答：第一个`compile`表示执行`build`时的默认`goal`，在idea中右键maven项目时会出现maven build选项，对应的也可以看到默认的goal加粗显示，表示执行的具体目标；第二个`compiler`为插件执行的触发条件，表示只有在项目执行`compile`时才会触发插件

----

> 判断在下方示例POM文件中，最终文件是否会被输出至targetPath

```
<project xmlns="http://maven.apache.org/POM/4.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                      https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <build>
    ...
    <resources>
      <resource>
        <targetPath>META-INF/plexus</targetPath>
        <filtering>false</filtering>
        <directory>${basedir}/src/main/plexus</directory>
        <includes>
          <include>*.properties</include>
        </includes>
        <excludes>
          <exclude>*.properties</exclude>
        </excludes>
      </resource>
    </resources>
    <testResources>
      ...
    </testResources>
    ...
  </build>
</project>
```

答：不会，exclude的优先级高于include


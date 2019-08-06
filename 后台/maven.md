# maven

## 概述

- 自动化开发工具
- 可以简化和标准化项目建设过程
- 处理编译，分配，文档，团队协作和其他任务的无缝连接。   
- pom.xml(Project Object Model)  

## 环境配置
- jdk
    - path
    - classpath
- maven
    - M2_HOME
    - MAVEN_HOME  

用`mvn -version`来测试是否正常工作

## 生命周期
maven的生命周期就是对所有构建过程抽象与统一，生命周期包含项目的清理、初始化、编译、测试、打包、集成测试、验证、部署、站点生成等几乎所有的过程。  
Maven有三套==相互独立==的生命周期：
- Clean Lifecycle（Clean生命周期）：在进行真正的构建之前进行一些清理工作。
  
    |    阶段    |                 功能                  |
    | :--------: | :-----------------------------------: |
    | pre-clean  |   执行一些需要在clean之前完成的工作   |
    |   clean    |     移除所有上一次构建生成的文件      |
    | post-clean | 执行一些需要在clean之后立刻完成的工作 |

- Default Lifecycle（Default生命周期）：构建的核心部分，编译，测试，打包，部署等等。
  
    |         阶段         |                             功能                             |
    | :------------------: | :----------------------------------------------------------: |
    |  process-resources   |            复制并处理资源文件至目标目录，准备打包            |
    |       compile        |                       编译项目的源代码                       |
    | process-test-sources |                 处理测试源码，比如过滤一些值                 |
    |         test         | 使用合适的单元测试框架运行测试。这些测试应该不需要代码被打包或发布 |
    |       package        |    将编译好的代码打包成可分发的格式，如JAR，WAR，或者EAR     |
    |       install        |       安装包至本地仓库，以备本地的其它项目作为依赖使用       |
    |        deploy        | 复制最终的包至远程仓库，共享给其它开发人员和项目（通常和一次正式的发布相关） |

- Site Lifecycle（Site生命周期）：生成项目报告，站点，发布站点。  

    |    阶段     |                            功能                            |
    | :---------: | :--------------------------------------------------------: |
    |  pre-site   |          执行一些需要在生成站点文档之前完成的工作          |
    |    site     |                     生成项目的站点文档                     |
    |  post-site  | 执行一些需要在生成站点文档之后完成的工作，并且为部署做准备 |
    | site-deploy |            将生成的站点文档部署到特定的服务器上            |
    

==在一个生命周期中，运行某个阶段的时候，它之前的所有阶段都会被运行==，也就是说，`mvn clean` 等同于`mvn pre-clean clean`，如果我们运行`mvn post-clean`，那么`pre-clean`，`clean`都会被运行。这是Maven很重要的一个规则，可以大大简化命令行的输入。

## 工程
### 创建工程
用`mvn archetype:generate`命令选择模板创建工程，通常情况下选择以下两个：
- maven-archetype-webapp – Java Web Project (WAR)
- maven-archetype-quickstart – Java Project (JAR)  

然后填写groupId，artifactId，version，package等标识  
要导入项目到eclipse中，需要`mvn eclipse:eclipse`

### 构建&&测试工程
- 编译项目：`mvn compile`
- 打包发布：`mvn package`
- 清理：`mvn clean`

## 配置文件

### settings.xml
- localRepository：表示构建系统本地仓库的路径，默认值是`${user.home}/.m2/repository`
- interactiveMode：表示maven是否需要和用户交互以获得输入，默认为true
- usePluginRegistry：表示maven是否需要使用plugin-registry.xml文件来管理插件版本，默认为false
- offline：表示maven是否需要在离线模式下运行，默认为false
- pluginGroups：当插件的groupId没有显式提供时，供搜寻插件groupId的列表
- servers：一些类似用户名、密码的信息保存在这里
- mirrors：为仓库列表配置的下载镜像列表

    ```xml
    <mirrors>
        <!-- 给定仓库的下载镜像 -->
        <mirror>
            <!-- 镜像的唯一标识符 -->
            <id />
            <!-- 镜像名称 -->
            <name />
            <!-- 镜像的url，构建系统会优先考虑该url，而非使用默认服务器的url -->
            <url />
            <!-- 被镜像服务器的id -->
            <mirrorOf />
        </mirror>
    </mirrors>
    ```
- proxies：用来配置不同的代理
- profiles：根据环境参数来调整构建配置的列表

    ```xml
    <profiles>
        <profile>
            <!-- profile的唯一标识 -->
            <id />
            <!-- 自动触发profile的条件逻辑 -->
            <activation />
            <!-- 扩展属性列表 -->
            <properties />
            <!-- 远程仓库列表 -->
            <reproperties />
            <!-- 插件仓库列表 -->
            <pluginRepositories />
        </profile>
    </profiles>
    ```
    - activation
    - properties
    - repositories：远程仓库列表
    
        ```xml
        <repositories>
            <!-- 包含需要连接到远程仓库的信息 -->
            <repository>
                <!-- 远程仓库唯一标识 -->
                <id />
                <!-- 远程仓库名称 -->
                <name />
                <!-- 如何处理远程仓库里发布版本的下载 -->
                <release>
                    <!-- true或false表示该仓库是否为下载某种类型的构件（发布版，快照版）开启。 -->
                    <enable />
                    <!-- 指定更新发生的频率。maven会比较本地pom与远程pom的时间戳。选项是：always（一直），daily（默认，每日），interval: X（X为以分钟为单位的时间间隔），never（从不） -->
                    <updatePolicy />
                    <!-- 当maven验证构件文件失败时应该怎么做：ignore，fail，warn -->
                    <checksumPolicy />
                </release>
                <!-- 如何处理远程仓库快照版本的下载 -->
                <snapshots>
                    <enable />
                    <updatePolicy />
                    <checksumPolicy />
                </snapshots>
                <!-- 远程仓库url，格式protocol://hostname/path -->
                <url />
                <!-- 用于定位和排序构件的仓库布局类型，default（默认），legacy（遗留） -->
                <layout />
            </repository>
        </repositories>
        ```
    - pluginRepositories：和repositories类似，只是repositories管理jar包依赖关系，pluginRepositories则是管理插件的仓库
- activeProfiles：手动激活profiles的列表，按照profile被应用的顺序定义activeProfiles

### pom.xml
每个工程应该只有一个POM文件。
- 所有的POM文件需要project元素和三个必须的字段：groupId, artifactId, version。
- 在仓库中的工程标识为groupId:artifactId:version
- POM.xml的根元素是project，它有三个主要的子节点：
    - groupId: 工程组标识
    - artifactId: 工程的标识
    - version: 版本的标识

```xml
<project>
    <modelVersion>4.0.0</modelVersion>
    <!-- 基础设置 -->
    <groupId>...</groupId>
    <artifactId>...</artifactId>
    <version>...</version>
    <packaging>...</packaging>
    <name>...</name>
    <url>...</url>
    <dependencies>...</dependencies>
    <parent>...</parent>
    <dependencyManagement>...</dependencyManagement>
    <modules>...</modules>
    <properties>...</properties>

    <!--构建设置 -->
    <build>...</build>
    <reporting>...</reporting>

    <!-- 更多项目信息 -->
    <name>...</name>
    <description>...</description>
    <url>...</url>
    <inceptionYear>...</inceptionYear>
    <licenses>...</licenses>
    <organization>...</organization>
    <developers>...</developers>
    <contributors>...</contributors>
    
    <!-- 环境设置 -->
    <issueManagement>...</issueManagement>
    <ciManagement>...</ciManagement>
    <mailingLists>...</mailingLists> 
    <scm>...</scm>
    <prerequisites>...</prerequisites>
    <repositories>...</repositories>
    <pluginRepositories>...</pluginRepositories>
    <distributionManagement>...</distributionManagement>
    <profiles>...</profiles>
</project>

```
## 仓库
- 本地（local）：是本地上的一个文件夹
- 中央（central）：Maven社区提供的仓库
- 远程（remote）：开发人员自己定制的仓库

## 插件

## 依赖关系
项目的依赖关系主要分为三种：依赖，继承，聚合 

### 依赖关系
依赖关系是最常用的一种，就是你的项目需要依赖其他项目，比如Apache-common包，Spring包等等。
```xml
<dependency>  
   <groupId>junit</groupId>  
   <artifactId>junit</artifactId>  
   <version>4.11</version>  
   <scope>test</scope>  
   <type>jar</type>  
   <optional>true</optional>  
</dependency>  
```

任意一个外部依赖说明包含如下几个要素：
- groupId（必须）
- artifactId（必须）
- version（必须）：可以用区间表达式来表示，比如(2.0, )表示>2.0，[2.0, 3.0)表示2.0<=ver<3.0；多个条件之间用逗号分隔，比如[1,3]，[5,7]。
- scope
    |          选项          |                             解释                             |
    | :--------------------: | :----------------------------------------------------------: |
    |  compile（编译范围）   | compile是默认的范围；编译范围依赖在所有的classpath中可用，同时它们也会被打包。 |
    | provided（已提供范围） | provided依赖只有在当JDK或者一个容器已提供该依赖之后才使用。  |
    | runtime（运行时范围）  | runtime依赖在运行和测试系统的时候需要，但在编译的时候不需要。 |
    |    test（测试范围）    | test范围依赖在编译和运行时都不需要，它们只有在测试编译和测试运行阶段可用。 |
    |   system（系统范围）   | system范围依赖与provided类似，但是你必须显式的提供一个对于本地系统中JAR文件的路径。 |
- type：一般不用配置，默认是jar。当type为pom时，代表引用关系。
- optional

### 继承关系


### 聚合关系
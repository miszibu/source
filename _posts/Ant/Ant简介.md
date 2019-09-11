---
title: Ant简介
date: 2019-03-10 13:56:54
tags: [Ant]
---

> 这里是Ant使用指南，根据目录，直接来查找如何编辑Ant所需要的XML文件。

<!--more-->

## Ant  Project

Apache Ant使用XML来编写构建文件，每个构建文件**包含一个Project**和**至少一个Default Target**。

*  **target**: 是一个任务，一个target就是一段实际可执行的代码。
* **Project**: 标注一个项目，设置默认Target，也可以标准basedir供后续使用

```xml
<project name="java-ant project" default="run" basedir=".">  
    ...  
</project>
```

项目(`project`)标记使用各种属性来设置要运行的名称和目标。最常用的属性如下所示。

| 属性      | 描述                                                  | 必需?  |
| --------- | ----------------------------------------------------- | ------ |
| `name`    | 这是该项目的名称                                      | 非必需 |
| `default` | 如果没有明确提供目标，它用于设置默认(`default`)目标。 | 非必需 |
| `basedir` | 它需要基目录路径                                      | 非必需 |

> 注意:可以选择要执行的目标。 如果没有给出目标，则使用项目的默认值。
>
> eg: ant xxx.xml xxTarget 
>
> 使用ant执行某个xml，调用其中某个target

------



## Ant Target

Target是一个或多个task的集合。每一个task都是一段可以执行的代码。 

目标可以依赖于其他目标，并且**依赖目标必须在当前目标之前执行**。

 在下方的例子中，一个run target 依赖于compile target。当我们执行run target时，会优先执行compile  target再继续run target。**可以依赖多个target。**

```xml
<target name="run" depends="compile">  
        ...  
</target>  
<target name="compile">  
        ...  
</target>
```

调用顺序:编译(compile)-> 运行(run)

> 注意:每个目标只执行一次，即使它有多个依赖目标。

目标具有以下列出的各种属性。

| 属性                      | 描述                                       | 必需？ |
| ------------------------- | ------------------------------------------ | ------ |
| `name`                    | 要设置目标的名称                           | 是     |
| `depends`                 | 它所依赖的目标列表。(可以依赖多个目标)     | 否     |
| `if`                      | 一个计算结果为`true`的属性                 | 否     |
| `unless`                  | 一个计算结果为`false`的属性                | 否     |
| `description`             | 这个目标函数的简短描述                     | 否     |
| `extensionOf`             | 将当前目标添加到扩展点的从属列表。         | 否     |
| `onMissingExtensionPoint` | 如果此目标扩展了缺少的扩展点，该如何处理。 | 否     |

------



## Ant Task

Apache Ant任务分为两类:

- 内置任务
- 用户定义的任务

```xml
<task-name attribute1 = "value1" attribute2 = "value2" ... >  
    ...  
</task-name>
```

### Apache Ant预定义(内置)任务

Apache Ant本身在其库中提供的任务称为内置任务。 Apache ant提供了大量内置任务，可用于执行区分任务。 如下列表所示:

- 存档任务
- 审计任务
- 编译任务
- 执行任务
- 文件任务
- 记录任务
- 邮件任务

#### 存档任务

用于压缩和解压缩数据的任务称为归档任务。下面列出了一些常见的内置存档任务。

| 任务名称 | 描述                              |
| -------- | --------------------------------- |
| Ear      | Jar任务的扩展，对文件进行特殊处理 |
| Jar      | 一组文件                          |
| Tar      | 创建tar存档                       |
| Unjar    | 解压缩jar文件                     |
| Untar    | 解压tarfile                       |
| Unwar    | 解压缩warfile                     |
| Unzip    | 解压缩zip文件                     |
| War      | Jar任务的扩展                     |

#### 审计任务

| 任务名称 | 描述                    |
| -------- | ----------------------- |
| JDepend  | 它用于调用JDepend解析器 |

#### 编译任务

用于编译源文件的任务称为编译任务，下面列出了一些常见的内置编译任务。

| 任务名称 | 描述                       |
| -------- | -------------------------- |
| Depend   | 确定哪些类文件的资源已过期 |
| Javac    | 编译源文件                 |
| JspC     | 运行JSP编译器              |
| NetRexxC | 编译NetRexx源文件          |
| Rmic     | 运行rmic编译器             |

#### 执行任务

用于执行运行应用程序的任务称为执行任务。下面列出了一些常见的内置执行任务。

| 任务名称 | 描述                             |
| -------- | -------------------------------- |
| Ant      | 在指定的构建文件上运行Ant        |
| AntCall  | 在同一个构建文件中运行另一个目标 |
| Apply    | 执行系统命令                     |
| Java     | 执行Java类                       |
| Parallel | 可包含其他ant任务的容器任务      |
| Sleep    | 按指定的时间暂停执行             |

#### 文件任务

与句柄文件操作相关的任务称为文件任务。下面列出了一些常见的内置文件任务。

| 任务名称 | 描述                 |
| -------- | -------------------- |
| Chmod    | 更改文件的权限       |
| Chown    | 更改文件的所有权     |
| Concat   | 连接多个文件         |
| Copy     | 将文件复制到新目的地 |
| Delete   | 删除文件             |
| Mkdir    | 创建一个目录         |

### 如何使用Apache Ant任务？

要使用任务，首先需要使用`<project>`标签创建项目。 之后，创建一个目标，使用`<target>`标记对任务进行分组。 然后可以通过将任务放在目标标记内来执行任务。看一个例子，这里使用`<java>`标签创建Java任务。

```xml
<project name="java-ant project" default="run">  
    <target name="run" depends="compile">  
        <java classname = "com.yiibai.Hello">  
            <classpath path="test"></classpath>  
        </java>  
    </target>  
</project>
```

### Apache Ant用户定义任务

Apache Ant允许用户编写自己的任务。编写自己的任务非常容易。 下面给出了一些必要的步骤。请参考以下几个步骤。

- 首先创建一个Java类并扩展`org.apache.tools.ant.Task`类。
- 为每个属性创建`setter`和`getter`方法。
- 如果`task`包含其他任务作为嵌套元素，则`class`必须实现`org.apache.tools.ant.TaskContainer`接口。
- 如果任务支持字符数据，请编写`public void addText(String)`方法。
- 对于每个嵌套元素，`write`，`add`或`addConfigured`方法。
- 编写一个`public void execute()`方法(不带参数)并抛出`BuildException`。

------



## Ant Property

Property是键值对。属性设置后可在构建文件中的任何位置访问的值。

Apache Ant属性类型有两种:

- 内置属性
- 用户定义的属性

### Apache Ant内置属性

| 属性                          | 描述                                     |
| ----------------------------- | ---------------------------------------- |
| `basedir`                     | 用于项目基础的绝对路径                   |
| `ant.file`                    | 用于构建文件的绝对路径                   |
| `ant.version`                 | 用于Ant的版本                            |
| `ant.project.name`            | 它包含当前正在执行的项目的名称           |
| `ant.project.default-target`  | 它包含当前正在执行的项目的默认目标的名称 |
| `ant.project.invoked-targets` | 调用当前项目时的目标列表                 |
| `ant.java.version`            | 拥有的JVM版本                            |
| `ant.core.lib`                | `ant.jar`文件的绝对路径                  |
| `ant.home`                    | 包含Ant的主目录                          |
| `ant.library.dir`             | 包含用于加载Ant的jar的目录。             |

### Apache Ant用户定义的属性。

**Apache Ant属性示例**

```xml
<project name="apache-ant project" default="run">  
    <property name="student-name" value = "Maxsu"></property>  
    <target name="run">  
        <echo>${student-name} is our student.</echo>  
    </target>  
    <target name="compile">  
        <javac includeantruntime="false" srcdir="./src" destdir = "test"></javac>  
    </target>  
</project>
```

------



## 相关资料

[易百教程 Ant](https://www.yiibai.com/ant/apache-ant-command-line-arguments.html)
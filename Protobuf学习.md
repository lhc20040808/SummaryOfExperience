# Protobuf



### Protobuf是什么

Protobuf是一种平台无关、语言无关、可扩展且轻便高效的序列化数据结构的协议，可以用于**网络通信**和**数据存储**。



### 为什么要使用Protobuf

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/protobuf_guide.png)

### 如何使用Protobuf

```java
protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/addressbook.proto
```

> -I 编译源文件的目录

> --java_out 编译目录文件

通过这个命令会自动编译出java代码，目前protobuf支持以下语言

|  Language   |                  Source                  |
| :---------: | :--------------------------------------: |
|     C++     | [src](https://github.com/google/protobuf/blob/master/src) |
|    Java     | [java](https://github.com/google/protobuf/blob/master/java) |
|   Python    | [python](https://github.com/google/protobuf/blob/master/python) |
| Objective-C | [objectivec](https://github.com/google/protobuf/blob/master/objectivec) |
|     C#      | [csharp](https://github.com/google/protobuf/blob/master/csharp) |
|  JavaNano   | [javanano](https://github.com/google/protobuf/blob/master/javanano) |
| JavaScript  | [js](https://github.com/google/protobuf/blob/master/js) |
|    Ruby     | [ruby](https://github.com/google/protobuf/blob/master/ruby) |
|     Go      | [golang/protobuf](https://github.com/golang/protobuf) |
|     PHP     | [php](https://github.com/google/protobuf/blob/master/php) |
|    Dart     | [dart-lang/protobuf](https://github.com/dart-lang/protobuf) |

由于命令行的方式编译代码非常繁琐，且效率极低。谷歌提供了开源的[Protobuf Gradle插件](https://github.com/google/protobuf-gradle-plugin)

简单说一下配置方式

在project.gradle中配置

```groovy
buildscript {
  repositories {
    mavenLocal()
  }
  dependencies {
    classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.6-SNAPSHOT'
  }
}
```

在modle.gradle中配置

```groovy
apply plugin: 'com.google.protobuf'

dependencies {
  // You need to depend on the lite runtime library, not protobuf-java
  compile 'com.google.protobuf:protobuf-lite:3.0.0'
}

protobuf {
  protoc {
    // You still need protoc like in the non-Android case
    artifact = 'com.google.protobuf:protoc:3.0.0'
  }
  plugins {
    javalite {
      // The codegen for lite comes as a separate artifact
      artifact = 'com.google.protobuf:protoc-gen-javalite:3.0.0'
    }
  }
  generateProtoTasks {
    all().each { task ->
      task.builtins {
        // In most cases you don't need the full Java output
        // if you use the lite output.
        remove java
      }
      task.plugins {
        javalite { }
      }
    }
  }
}
```

目前有Protobuf2和Protobuf3，本文以Protobuf2为例，简单介绍一下Protobuf2的语法，更多详细内容请参考[官方文档](https://developers.google.com/protocol-buffers/docs/proto)(需要翻墙)

先在Java的同级目录下新建一个名为proto的文件夹专门用于存放proto文件，编写proto文件后编译模块会根据proto文件内容生成java文件。

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/protobuf_use.png)

来看一下名为Test.proto的文件内容

```protobuf
//指定protobuf语法版本
syntax = "proto2";

//包名
option java_package = "com.lhc.protobuf";
//源文件类名
option java_outer_classname = "AddressBookProtos";

// class Person
message Person {
  //required 必须设置（不能为null）
  required string name = 1;
  //int32 对应java中的int
  required int32 id = 2;
  //optional 可以为空
  optional string email = 3;

  enum PhoneType {
    MOBILE = 0;
    HOME = 1;
    WORK = 2;
  }

  message PhoneNumber {
    required string number = 1;
    optional PhoneType type = 2 [default = HOME];
  }
   //repeated 重复的 （集合）
  repeated PhoneNumber phones = 4;
}

message AddressBook {
  repeated Person people = 1;
}
```



### Protobuf应用------网络传输

##### http传输

通常在应用层我们使用的都是Http协议，Http的本质是一次socket请求的连接与断开。传输数据时将protobuf对象转换为byte[]传输即可

##### 自定义TCP通信协议

当我们自定义TCP通信协议的时候，将面临粘包与分包的问题

分包：

- 要发送的数据大于TCP缓冲剩余空间


- 待发送数据大于MSS（最大报文长度）

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/http_unpack.png)

粘包：

- 要发送的数据小于TCP缓冲区，将多次写入缓冲区的数据一起发送
- 接收端的应用层没有及时读取缓冲区的数据

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/http_stick.png)



自定义通信协议的两种方式

- 定义数据包包头

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/defined_tcp_1.png)

- 在数据包之间设置边界

![](https://raw.githubusercontent.com/lhc20040808/Pictures/master/res/图片/defined_tco_2.png)

大家可以参考 JT808协议 ------交通部808协议（车联网），也是采用类似的方式定义通信协议



### 手写简易Gradle Protobuf编译插件

准备proto编译器工件，proto文件目录，通过参数拼接出命令行编译proto文件，将执行结果注册到编译打包列表

定义两个DSL命名空间

```groovy
class ProtobufExt {
     /**
     * proto文件目录
     */
    def srcDirs

    ProtobufExt() {
        srcDirs = []
    }

    def srcDir(String srcDir) {
        if (!srcDirs.contians(srcDir))
            srcDirs << srcDir
    }

    def srcDir(String... srcDirs) {
        srcDirs.each { srcDir(it) }
    }
}
```

```groovy
class ProtoExt {
    def path
    def artifact
}
```



定义一个插件实现`Plugin`接口

```groovy
class ProtobufPlugin implements Plugin<Project> {

    static final String PROTOBUF_EXTENSION_NAME = "protobuf"
    static final String PROTO_SUB_EXTENSION_NAME = "protoc"
    Project project

    @Override
    void apply(Project project) {
        this.project = project
        project.apply plugin: 'com.google.osdetector'
        project.extensions.create(PROTOBUF_EXTENSION_NAME, ProtobufExt)//创建命名空间
        project.protobuf.extensions.create(PROTO_SUB_EXTENSION_NAME, ProtoExt)
        //在gradle分析之后执行
        project.afterEvaluate {
            if (!project.protobuf.protoc.path) {
                if (!project.protobuf.protoc.artifact) {
                    throw new GradleException("请配置protoc编译器")
                }
                //创建依赖配置
                Configuration config = project.configurations.create("protobufConfig")
                def (group, name, version) = project.protobuf.protoc.artifact.split(":")
                def notation = [group: group, name: name, version: version, classifier: project.osdetector.classifier, ext: 'exe']
                //本地存在则返回工件，否则先下载
                Dependency dependency = project.dependencies.add(config.name, notation)
                //获得对应dependency的所有文件
                File file = config.fileCollection(dependency).singleFile
                println file
                if (!file.canExecute() && !file.setExecutable(true)) {
                    throw new GradleException("protoc编译器无法执行")
                }
                project.protobuf.protoc.path = file.path
            }

            Task task = project.tasks.create("compileProtobuf", CompileProtobufTask)
            task.inputs.files(project.protobuf.srcDirs)
            task.outputs.dir("${project.buildDir}/generated/source/proto")

            //将编译生成的java文件假如到工程源代码文件列表中
            linkProtoToJavaSource()
        }
    }

    /**
     * 判断是否为安卓工程
     * @return
     */
    boolean isAndroidProject() {
        return project.plugins.hasPlugin(AppPlugin) || project.plugins.hasPlugin(LibraryPlugin)
    }

    def getAndroidVariants() {
        return project.plugins.hasPlugin(AppPlugin) ?
                project.android.applicationVariants + project.android.testVariants : project.android.libraryVariants + project.android.testVariants
    }

    def linkProtoToJavaSource() {
        if (isAndroidProject()) {
            androidVariants.each {
                BaseVariant variant ->
                    //将任务加入构建过程,并将第二个参数的文件注册到编译列表当中
                    variant.registerJavaGeneratingTask(project.tasks.compileProtobuf, project.tasks.compileProtobuf.outputs.files.files)
            }
        } else {
            project.sourceSets.each {
                SourceSet sourceSet ->
                    def compileName = sourceSet.getCompileTaskName('java')
                    JavaCompile javaCompile = project.tasks.getByName(compileName)
                    javaCompile.dependsOn project.tasks.compileProtobuf
                    sourceSet.java.srcDirs(project.tasks.compileProtobuf.outputs.files.files)
            }
        }
    }
}
```

实现一个`DefaultTask`子类，主要是通过输入参数拼接出如下的编译所需的命令行

`protoc -I=$SRC_DIR --java_out=$DST_DIR $SRC_DIR/addressbook.proto`

```
class CompileProtobufTask extends DefaultTask {

    CompileProtobufTask() {
        group = 'Protobuf'
        outputs.upToDateWhen { false } //关闭增量构建，否则输入输出不变时执行增量构建
    }
    
    @TaskAction
    def run() {
        def outDir = outputs.files.singleFile
        outDir.deleteDir()
        outDir.mkdirs()

        def cmd = [project.protobuf.protoc.path]

        cmd << "--java_out=$outDir"
        def source = []
        def inDirs = inputs.files.files
        inDirs.each {
            cmd << "-I=${it.path}"
        }

        getProtoFiles(inDirs, source)

        cmd.addAll(source)
        println "执行:$cmd"

        Process process = cmd.execute()

        def stdout = new StringBuffer()
        def stdErr = new StringBuffer()

        process.waitForProcessOutput(stdout, stdErr)//输出错误日志
        if (process.exitValue() == 0) {
            println "编译protobuf文件成功"
        } else {
            throw new GradleException("编译protobuf文件失败" + " $stdout" + " $stdErr")
        }

    }

    /**
     * 将目录下所有.proto文件添加到集合
     * @param dirs
     * @param source
     */
    def getProtoFiles(dirs, source) {
        dirs.each {
            File file ->
                if (file.isDirectory()) {
                    getProtoFiles(file.listFiles(), source)
                } else if (file.name.endsWith(".proto")) {
                    source << file
                }
        }
    }
}
```


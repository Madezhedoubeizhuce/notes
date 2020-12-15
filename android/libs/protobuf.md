# Android ProtoBuf应用及原理

ProtoBuf是一种灵活高效的可序列化的数据协议，它比xml和json格式数据更加高效。

## 接入



在项目的根gradle配置如下：

```groovy
dependencies {
	classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.5'
}
```

在gradle中配置如下：

```groovy
apply plugin: 'com.google.protobuf'

android {
    sourceSets {
        main {
            // 定义proto文件目录
            proto {
                srcDir 'src/main/proto'
                include '**/*.proto'
            }
        }
    }
}

dependencies {
    // 定义protobuf依赖，使用精简版
    implementation "com.google.protobuf:protobuf-lite:3.0.0"
}

protobuf {
    generatedFilesBaseDir = "$projectDir/src/generated"
    protoc {
        artifact = 'com.google.protobuf:protoc:3.0.0'
    }
    plugins {
        javalite {
            artifact = 'com.google.protobuf:protoc-gen-javalite:3.0.0'
        }
    }
    generateProtoTasks {
        all().each { task ->
            task.plugins {
                javalite {}
            }
        }
    }
}
```

`apply plugin: 'com.google.protobuf'`是Protobuf的Gradle插件，帮助我们在编译时通过语义分析自动生成源码，提供数据结构的初始化、序列化以及反序列等接口。

`compile "com.google.protobuf:protobuf-lite:3.0.0"`是Protobuf支持库的精简版本，在原有的基础上，用public替换set、get方法，减少Protobuf生成代码的方法数目。

## 定义数据结构

test.proto定义如下：

```protobuf
syntax = "proto3";
package com.alpha.test;
option java_outer_classname = "ProtoTest";

message Test {
  string msg = 1;
  int32 num = 2;
  int32 page = 3;
}
```

编译后的类结构如下：

![image-20201215224222457](.\imgs\image-20201215224222457.png)

![image-20201215224317931](.\imgs\image-20201215224317931.png)

可以看到生成的类中提供了Builder、writeTo以及一系列的parse方法，用以解析二进制数据以及将ProtoBuf消息封装成二进制数据。

## 编码

先写个简单的demo，主要功能就是创建一个Test对象，然后将这个对象编码成二进制数据：

```kotlin
val testMsg = ProtoTest.Test.newBuilder()
            .setMsg("123456")
            .setNum(56)
            .setPage(2)
            .build()

val file = File("./test/ProtoTest.bin")
if (file.parentFile?.exists() == false) {
    file.parentFile?.mkdirs()
}
val os = FileOutputStream(file)
testMsg.writeTo(os)
```

代码中先创建了一个Test的对象，并且将msg设为`123456`，num设为`23`，page设为`2`，它生成的二进制编码如下：

```
0a 06 31 32 33 34 35 36 10 38 18 02
```

为了理解这段二进制编码，我们需要了解以下这些知识：

### 消息结构

protocol buffer消息是由一系列的键值对组成，并且使用字段的序号作为键，
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


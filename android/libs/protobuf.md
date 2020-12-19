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

## 基本使用

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

### 数据类型

protobuf提供如下数据类型：

| .proto Type | Notes                                                        | C++ Type | Java Type  | Python Type[2] | Go Type | Ruby Type                      | C# Type    | PHP Type          | Dart Type |
| :---------- | :----------------------------------------------------------- | :------- | :--------- | :------------- | :------ | :----------------------------- | :--------- | :---------------- | :-------- |
| double      |                                                              | double   | double     | float          | float64 | Float                          | double     | float             | double    |
| float       |                                                              | float    | float      | float          | float32 | Float                          | float      | float             | double    |
| int32       | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint32 instead. | int32    | int        | int            | int32   | Fixnum or Bignum (as required) | int        | integer           | int       |
| int64       | Uses variable-length encoding. Inefficient for encoding negative numbers – if your field is likely to have negative values, use sint64 instead. | int64    | long       | int/long[3]    | int64   | Bignum                         | long       | integer/string[5] | Int64     |
| uint32      | Uses variable-length encoding.                               | uint32   | int[1]     | int/long[3]    | uint32  | Fixnum or Bignum (as required) | uint       | integer           | int       |
| uint64      | Uses variable-length encoding.                               | uint64   | long[1]    | int/long[3]    | uint64  | Bignum                         | ulong      | integer/string[5] | Int64     |
| sint32      | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int32s. | int32    | int        | int            | int32   | Fixnum or Bignum (as required) | int        | integer           | int       |
| sint64      | Uses variable-length encoding. Signed int value. These more efficiently encode negative numbers than regular int64s. | int64    | long       | int/long[3]    | int64   | Bignum                         | long       | integer/string[5] | Int64     |
| fixed32     | Always four bytes. More efficient than uint32 if values are often greater than 228. | uint32   | int[1]     | int/long[3]    | uint32  | Fixnum or Bignum (as required) | uint       | integer           | int       |
| fixed64     | Always eight bytes. More efficient than uint64 if values are often greater than 256. | uint64   | long[1]    | int/long[3]    | uint64  | Bignum                         | ulong      | integer/string[5] | Int64     |
| sfixed32    | Always four bytes.                                           | int32    | int        | int            | int32   | Fixnum or Bignum (as required) | int        | integer           | int       |
| sfixed64    | Always eight bytes.                                          | int64    | long       | int/long[3]    | int64   | Bignum                         | long       | integer/string[5] | Int64     |
| bool        |                                                              | bool     | boolean    | bool           | bool    | TrueClass/FalseClass           | bool       | boolean           | bool      |
| string      | A string must always contain UTF-8 encoded or 7-bit ASCII text, and cannot be longer than 232. | string   | String     | str/unicode[4] | string  | String (UTF-8)                 | string     | string            | String    |
| bytes       | May contain any arbitrary sequence of bytes no longer than 232. | string   | ByteString | str            | []byte  | String (ASCII-8BIT)            | ByteString | string            | List      |

## 编码

先写个简单的demo，主要功能就是创建一个Test对象，然后将这个对象编码成二进制数据：

```kotlin
val testMsg = ProtoTest.Test.newBuilder()
            .setMsg("12345678")
            .setNum(240)
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
0a 08 31 32 33 34 35 36 37 38 10 f0 01 18 02  
```

为了理解这段二进制编码，请继续往下读。

### key编码规则

protocol buffer消息是由一系列的key-value组成，如test.proto中所定义，并且每个字段都有对应的序号，在编码时使用序号对key进行编码。

编码后的key实际包括两部分：proto文件中定义的字段序号和代表value长读的字段类型（wire_type），所有字段类型如下：

| Type | Meaning          | Used For                                                 |
| :--- | :--------------- | :------------------------------------------------------- |
| 0    | Varint           | int32, int64, uint32, uint64, sint32, sint64, bool, enum |
| 1    | 64-bit           | fixed64, sfixed64, double                                |
| 2    | Length-delimited | string, bytes, embedded messages, packed repeated fields |
| 3    | Start group      | groups (deprecated)                                      |
| 4    | End group        | groups (deprecated)                                      |
| 5    | 32-bit           | fixed32, sfixed32, float                                 |

key的具体编码规则如下：

```
(field_number << 3) | wire_type
```

现在来看看Test中各字段key的编码：

首先，msg的序号是1，并且是string类型，即wire_type是2，按照key的编码规则的到如下公式：

```
(0x01 << 3) | 0x02
```

计算结果是0x0a，也就是编码后二进制数据中的第一个0a。

然后就是num，它的序号是2，且类型int32类型，因此可以得到下面的公式：

```
(0x02 << 3) | 0
```

计算结果就是0x10，对应上面二进制数据的倒数第四个字节。

最后是page，它的计算方式和num一样，最后的结果是0x18。

### value编码规则

#### Base 128 Varints 编码

Base 128 Varints编码基于一个现实：**越小的数字，越经常使用**

举个例子：

如果我们要传输数字1，假设使用32为整形，那么传输的二进制数据就是：

```
00000000 00000000 00000000 00000001
```

可以看到这里面几乎所有数据都是0，也就是这里面几乎都是无效数据。Base 128 Varints编码就是要解决这个事情，下面介绍一下它的编码规则：

1. 将原始数据有效位分组，从低到高每七位分为一组
2. 将所有分好的组排序，按照低位组在前，高位组在后的规则排序
3. 给每个组加上最高位，构成一个字节，除最后一组外，其余组最高位都设为1，表示后面还有字节出现。最后一组最高位设为0，表示后面没有字节了。
4. 所有组的数据就构成了编码后的二进制数据。

是举两个例子来说明：

1. 一个字节，假设数字是 0000 0001
   1. 分组：因为只有一个字节，所以只有一个分组，000 0001
   2. 排序：一个分组排序结果就是它本身：000 0001
   3. 加上最高位：因为只有一个分组，所以最高位加上0：0000 0001
   4. 最终得到的编码还是它本身：000 0001
2. 多个字节，假设数字是 0110 1011 0110 0011
   1. 分组：将有效为从低到高每七位分组，得到如下结果
      - 110 1011
      -  101 0110
      -  000 0001
   2. 排序，按照低位组在前，高位组在后的规则排序，得到如下结果：
      - 110 1011
      - 101 0110
      - 000 0001
   3. 加上最高位：
      - 1110 1011
      - 1101 0110
      - 0000 0001
   4. 最终的编码结果就是：
      - 1110 1011 1101 0110 0000 0001

#### string类型

string类型的value是由经过Base 128 Varints编码后的字符串长度后跟着字符串的byte值组成的。前面例子的中的字符串`12345678`经过编码后的值为：

```
08 31 32 33 34 35 36 37 38
```

开头的08表示当前字符串的长度，后面紧跟着`12345678`的byte值。

#### 无符号整数

无符号整数直接使用Base 128 Varints编码对字段值进行编码，如上面代码中的num为240，其对应的编码是：

```
f0 01
```

#### 有符号整数

对于正整数来说，使用Base 128 Varints编码能够压缩大量冗余数据，但是对于负整数来说并非如此。

一般计算机都用补码来表示整数，假设我们要计算-1的base 128 varints编码，-1的补码为：`11111111 11111111 11111111 11111111` 从这里就可以看出，直接对负数计算varints编码的话不仅不能压缩数据，反而会增加数据量。因此这里就要对负数在做一次转换，将其转为正数之后进行Base 128 Varints编码，这个转换就是**ZigZag编码**。

**ZigZag编码**的核心思想如下：

1. ***符号位放到最后，其他位整体前移一位***：
2. ***对于负数，把所有数据位按位求反，符号位保持不变***
3. ***对于正数，所有数据位按位求反***

对于32为有符号数整数，其实现如下：

```c
(n << 1) ^ (n >> 31)
```

对于64有符号整数：

```c
(n << 1) ^ (n >> 63)
```

经过ZigZag编码后，0还是0 ，-1变成了1，1变成了2：

| Signed Original | Encoded As |
| :-------------- | :--------- |
| 0               | 0          |
| -1              | 1          |
| 1               | 2          |
| -2              | 3          |
| 2147483647      | 4294967294 |
| -2147483648     | 4294967295 |

将有符号整数经过**ZigZag编码**之后就可以使用Base 128 Varints编码进行处理了。

#### Non-varint数字

Non-varint数字就是fixed64, sfixed64, double和fixed32, sfixed32, float类型的数字，他们的编码后的value就是个原始的二进制编码，其中fixed64, sfixed64, double长度为64位，fixed32, sfixed32, float长度为32位。

我们再Test中新增一个size字段，表示double类型：

```protobuf
message Test {
  string msg = 1;
  int32 num = 2;
  int32 page = 3;
  double size = 4;
}
```

设置size值为34：得到size字段对应的二进制编码为：

```
21 00 00 00 00 00 00 41 40  
```

#### 内嵌类型

内嵌消息的编码规则和string类型的编码规则是相同的，我们定义Test2:

```protobuf
message Test2 {
  Test test = 1;
}
```

使用如下代码保存其生成的二进制数据：

```kotlin
val testMsg = ProtoTest.Test.newBuilder()
.setMsg(
    "12345678"
)
.setNum(240)
.setPage(2)
.build()
val test2Msg = ProtoTest.Test2.newBuilder()
.setTest(testMsg)
.build()
val file = File("./test/ProtoTest2.bin")
if (file.parentFile?.exists() == false) {
    file.parentFile?.mkdirs()
}
val os = FileOutputStream(file)
test2Msg.writeTo(os)
```

得到如下数据：

```
0A 0F 0A 08 31 32 33 34 35 36 37 38 10 F0 01 18 02
```

可以看到，除了开头的0A 0F，后面的数据与之前的例子生成的数据是完全一致的。开头的0A和0F的计算方式和前面介绍string类型是一样的，这里就不再赘述了。

#### repeat类型

在proto2中，没有显式声明**[packed=true]**的repeat元素，其编码后的数据包含0个或多个相同key的key-value对，但是它们不一定是连续的，可能会交错分布在其他字段中，但是元素的顺序是有保证的。在proto3中，只用packed编码：

packed编码将所有的元素打包到一个单独的key-value对中，并且key的wire-type为2（key编码规则表），并且将每个元素按照它们的编码方式进行编码。在proto2中，需要在使用packed编码的字段后加上**[packed=true]**，在proto3中数值类型自动使用packed编码（？经试验默认并不是packed编码，而是连续的键值对，每个键都相同）。

假设定义如下message：

```protobuf
message Test2 {
  repeated int32 id = 2[packed = true];
}
```

那么他编码后的数据为：

```
12 04 01 02 03 04
```

可以看出，这和string类型的编码方式相同。

#### optional类型

proto3中的非repeat字段或者proto2中的optional类型字段可能有0个或者1个键值对。

在编码后的消息体重，可能出现多个重复的字段。如果是数值或者string类型，当出现多个相同的字段时，解析的时候应该使用最后出现的哪个字段。而对于内嵌类型，则需要把多个相同字段的值合并起来，类似MergeFrom方法，将多个对象种不同的属性合并。

#### 字段顺序

protobuf不能保证序列化时的字段顺序，因此两次序列化后的二进制数据可能不相同。
下一代Android渠道打包工具
========

## 最新版本

- **v1.0.4 - 2016.01.19** - 完善获取APK路径的方法,增加MarketInfo
- **v1.0.3 - 2016.01.14** - 增加缓存，新增ResUtils，更有好的错误提示
- **v1.0.2 - 2015.12.04** - 兼容productFlavors，完善异常处理
- **v1.0.1 - 2015.12.01** - 如果没有读取到渠道，默认返回空字符串
- **v1.0.0 - 2015.11.30** - 增加Java和Python打包脚本，增加文档
- **v0.9.9 - 2015.11.26** - 测试版发布，支持全新的极速打包方式 

## 项目介绍

[**packer-ng-plugin**](https://github.com/mcxiaoke/packer-ng-plugin) 是下一代Android渠道打包工具Gradle插件，支持极速打包，**1000**个渠道包只需要**5**秒钟，速度是 [**gradle-packer-plugin**](https://github.com/mcxiaoke/gradle-packer-plugin) 的**1000**倍以上，可方便的用于CI系统集成，支持自定义输出目录和最终APK文件名，依赖包： `com.mcxiaoke.gradle:packer-ng:1.0.+` 简短名：`packer`，可以在项目的 `build.gradle` 中指定使用，还提供了命令行独立使用的Java和Python脚本。实现原理见本文末尾。

## 使用指南

[`Maven Central`](http://search.maven.org/#search%7Cga%7C1%7Ca%3A%22packer-ng%22)

### 修改项目根目录的 `build.gradle`

```groovy

buildscript {
	......
	dependencies{
	// add packer-ng
		classpath 'com.mcxiaoke.gradle:packer-ng:1.0.4'
	}
}  
```

### 修改Android模块的 `build.gradle`

```groovy
apply plugin: 'packer' 

dependencies {
	// add packer-helper
	compile 'com.mcxiaoke.gradle:packer-helper:1.0.4'
} 
```

**注意：`packer-ng` 和 `packer-helper` 的版本号需要保持一致**

### Java代码中获取当前渠道

提示：`PackerNg.getMarket(Context)`内部缓存了结果，不会重复解析APK文件

```java

// 如果没有使用PackerNg打包添加渠道，默认返回的是""
// com.mcxiaoke.packer.helper.PackerNg
final String market = PackerNg.getMarket(Context)
// 或者使用 PackerNg.getMarket(Context,defaultValue)
// 之后就可以使用了，比如友盟可以这样设置
AnalyticsConfig.setChannel(market)

```

### 渠道打包脚本

可以通过两种方式指定 `market` 属性，根据需要选用：

- 打包时命令行使用 `-Pmarket= yourMarketFilePath` 指定属性
- 在 `gradle.properties` 里加入 `market=yourMarketFilePath`

market是你的渠道名列表文件，market文件是基于**项目根目录**的 `相对路径` ，假设你的项目位于 `~/github/myapp` 你的market文件位于 `~/github/myapp/config/markets.txt` 那么参数应该是 `-Pmarket=config/markets.txt`，一般建议直接放在项目根目录，如果market文件参数错误或者文件不存在会抛出异常。

渠道名列表文件是纯文本文件，每行一个渠道号，列表解析的时候会自动忽略空白行和格式不规范的行，请注意看命令行输出，渠道名和注释之间用 `#` 号分割开，可以没有注释，示例：

```
 Google_Play#play store market
 Gradle_Test#test
 SomeMarket#some market
 HelloWorld
```

渠道打包的Gradle命令行参数格式示例（在项目根目录执行）：  

```shell
./gradlew -Pmarket=markets.txt clean apkRelease
``` 

打包完成后你可以在 `${项目根目录}/build/archives/` 目录找到最终的渠道包。

#### 任务说明

渠道打包的Gradle Task名字是 `apk${buildType}` buildType一般是release，也可以是你自己指定的beta或者someOtherType，使用时首字母需要大写，例如release的渠道包任务名是 `apkRelease`，beta的渠道包任务名是 `apkBeta`，其它的以此类推。

#### 注意事项

**不支持`productFlavors`中定义的条件编译变量，不支持修改AndroidManifest**

如果你的项目有多个`productFlavors`，默认只会用第一个`flavor`生成的APK文件作为打包工具的输入参数，忽略其它`flavor`生成的apk，代码里用的是 `ariant.outputs[0].outputFile`。如果你想指定使用某个flavor来生成渠道包，可以用 `apkFlavor1Release`，`apkFlavor2Beta`这样的名字，示例（假设flavor名字是Intel）：

```shell
./gradlew -Pmarket=markets.txt clean apkIntelRelease
``` 

### 插件配置说明（可选） 

```groovy 
packer {
    // 指定渠道打包输出目录
    // archiveOutput = file(new File(project.rootProject.buildDir.path, "archives"))
    // 指定渠道打包输出文件名格式
    // 默认是 `${appPkg}-${flavorName}-${buildType}-v${versionName}-${versionCode}`
    // archiveNameFormat = ''
}
```

举例：假如你的App包名是  `com.your.company` ，渠道名是 `Google_Play` ，`buildType` 是 `release` ，`versionName` 是 `2.1.15` ，`versionCode` 是 `200115` ，那么生成的APK的文件名是 `com.your.company-Google_Player-release-2.1.15-20015.apk`   

* **archiveOutput**  指定渠道打包输出的APK存放目录，默认位于`${项目根目录}/build/archives`   

* **archiveNameFormat** - `Groovy格式字符串`， 指定渠道打包输出的APK文件名格式，默认文件名格式是： `${appPkg}-${flavorName}-${buildType}-v${versionName}-${versionCode}`，可使用以下变量:  
  
  * *projectName* - 项目名字
  * *appName* - App模块名字
  * *appPkg* - `applicationId` (App包名packageName)
  * *buildType* - `buildType` (release/debug/beta等)
  * *flavorName* - `flavorName` (对应渠道打包中的渠道名字)
  * *versionName* - `versionName` (显示用的版本号)
  * *versionCode* - `versionCode` (内部版本号)
  * *buildTime* - `buildTime` (编译构建日期时间)  

### 命令行打包脚本

如果不想使用Gradle插件，这里还有两个命令行打包脚本，在项目的 `tools` 目录里，分别是 `ngpacker-x.x.x-capsule.jar` 和 `ngpacker.py`，使用命令行打包工具，在Java代码里仍然是使用`packer-helper`包里的 `PackerNg.getMarket(Context)` 读取渠道

#### Java脚本

```shell
java -jar ngpacker-x.x.x-capsule.jar release_apk_file market_file
// help: java -jar packer-ng-x.x.x-capsule.jar
```

#### Python脚本

```shell
python ngpacker.py [file] [market] [output] [-h] [-s] [-t TEST]
// help: python packer-ng.py -h
// python; import ngpacker; help(ngpacker)
```

#### 不使用Gradle
使用命令行打包脚本，不想添加Gradle依赖的，可以完全忽略Gradle的配置，直接复制 [PackerNg.java](helper/src/main/java/com/mcxiaoke/packer/helper/PackerNg.java) 到项目中使用即可

## 实现原理

### PackerNg原理

#### 优点

- 使用APK注释字段保存渠道信息和MAGIC字节，从文件末尾读取渠道信息，速度快
- 实现为一个Gradle Plugin，支持定制输出APK的文件名等信息，方便CI集成
- 提供Java版和Python的独立命令行脚本，不依赖Gradle插件，支持独立使用
- 由于打包速度极快，单个包只需要5毫秒左右，可用于网站后台动态生成渠道包

#### 缺点

- 没有使用Android的productFlavors，无法利用flavors条件编译的功能

### 文件格式

Android应用使用的APK文件就是一个带签名信息的ZIP文件，根据 [ZIP文件格式规范](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT)，每个ZIP文件的最后都必须有一个叫 [Central Directory Record](https://users.cs.jmu.edu/buchhofp/forensics/formats/pkzip.html) 的部分，这个CDR的最后部分叫"end of central directory record"，这一部分包含一些元数据，它的末尾是ZIP文件的注释。注释包含**Comment Length**和**File Comment**两个字段，前者表示注释内容的长度，后者是注释的内容，正确修改这一部分不会对ZIP文件造成破坏，利用这个字段，我们可以添加一些自定义的数据，PackerNg项目就是在这里添加和读取渠道信息。

### 细节处理

原理很简单，就是将渠道信息存放在APK文件的注释字段中，但是实现起来遇到不少坑，测试了好多次。

#### ZipOutputStream.setComment

```java

FileOutputStream is = new FileOutputStream("demo.apk", true);
ZipOutputStream zos = new ZipOutputStream(is);
zos.setComment("Google_Market");
zos.finish();
zos.close();

ZipFile zipFile=new ZipFile("demo.apk");
System.out.println(zipFile.getComment());

```
使用Java写入APK文件注释虽然可以正常读取，但是安装的时候会失败，错误信息是：

```shell
adb install -r demo.apk
Failure [INSTALL_FAILED_INVALID_APK]
```

原因未知，可能Java的Zip实现写入了某些特殊字符导致APK文件校验失败，于是只能放弃这个方法。同样的功能使用Python测试完全没有问题，处理后的APK可以正常安装。

#### ZipFile.getComment

上面是ZIP文件注释写入，使用Java会导致APK文件被破坏，无法安装。这里是读取ZIP文件注释的问题，Java 7里可以使用 `zipFile.getComment()` 方法直接读取注释，非常方便。但是Android系统直到API 19，也就是4.4以上的版本才支持 [`ZipFile.getComment()`](http://developer.android.com/intl/zh-cn/reference/java/util/zip/ZipFile.html#getComment()) 方法。由于要兼容之前的版本，所以这个方法也不能使用。

#### 解决方法

由于使用Java直接写入和读取ZIP文件的注释都不可行，使用Python又不方便与Gradle系统集成，所以只能自己实现注释的写入和读取。实现起来也不复杂，就是为了提高性能，避免读取整个文件，需要在注释的最后加入几个MAGIC字节，这样从文件的最后开始，读取很少的几个字节就可以定位渠道名的位置。

几个常量定义：

```java
// ZIP文件的注释最长65535个字节
static final int ZIP_COMMENT_MAX_LENGTH = 65535;
// ZIP文件注释长度字段的字节数
static final int SHORT_LENGTH = 2;
// 文件最后用于定位的MAGIC字节
static final byte[] MAGIC = new byte[]{0x21, 0x5a, 0x58, 0x4b, 0x21}; //!ZXK!

```

#### 读写注释

Java版详细的实现见 [PackerNg.java](helper/src/main/java/com/mcxiaoke/packer/helper/PackerNg.java)，Python版的实现见 [ngpacker.py](tools/ngpacker.py) 。

写入ZIP文件注释：

```java

public static void writeZipComment(File file, String comment) 
throws IOException {
    byte[] data = comment.getBytes(UTF_8);
    final RandomAccessFile raf = new RandomAccessFile(file, "rw");
    raf.seek(file.length() - SHORT_LENGTH);
    // write zip comment length
    // (content field length + length field length + magic field length)
    writeShort(data.length + SHORT_LENGTH + MAGIC.length, raf);
    // write content
    writeBytes(data, raf);
    // write content length
    writeShort(data.length, raf);
    // write magic bytes
    writeBytes(MAGIC, raf);
    raf.close();
}

```

读取ZIP文件注释，有两个版本的实现，这里使用的是 `RandomAccessFile` ，另一个版本使用的是 `MappedByteBuffer` ，经过测试，对于特别长的注释，使用内存映射文件读取性能要稍微好一些，对于特别短的注释（比如渠道名），这个版本反而更快一些。

```java

public static String readZipComment(File file) throws IOException {
    RandomAccessFile raf = null;
    try {
        raf = new RandomAccessFile(file, "r");
        long index = raf.length();
        byte[] buffer = new byte[MAGIC.length];
        index -= MAGIC.length;
        // read magic bytes
        raf.seek(index);
        raf.readFully(buffer);
        // if magic bytes matched
        if (isMagicMatched(buffer)) {
            index -= SHORT_LENGTH;
            raf.seek(index);
            // read content length field
            int length = readShort(raf);
            if (length > 0) {
                index -= length;
                raf.seek(index);
                // read content bytes
                byte[] bytesComment = new byte[length];
                raf.readFully(bytesComment);
                return new String(bytesComment, UTF_8);
            }
        }
    } finally {
        if (raf != null) {
            raf.close();
        }
    }
    return null;
}

```

读取APK文件，由于这个库 `packer-helper` 需要同时给Gradle插件和Android项目使用，所以不能添加Android相关的依赖，但是又需要读取自身APK文件的路径，使用反射实现：

```java

// for android code
private static String getSourceDir(final Object context)
        throws ClassNotFoundException,
        InvocationTargetException,
        IllegalAccessException,
        NoSuchFieldException,
        NoSuchMethodException {
    final Class<?> contextClass = Class.forName("android.content.Context");
    final Class<?> applicationInfoClass = Class.forName("android.content.pm.ApplicationInfo");
    final Method getApplicationInfoMethod = contextClass.getMethod("getApplicationInfo");
    final Object appInfo = getApplicationInfoMethod.invoke(context);
    final Field sourceDirField = applicationInfoClass.getField("sourceDir");
    return (String) sourceDirField.get(appInfo);
}

```

#### Gradle Plugin

这个和旧版插件基本一致，首先是读取渠道列表文件，保存起来，打包的时候遍历列表，复制生成的APK文件到临时文件，给临时文件写入渠道信息，然后复制到输出目录，文件名可以使用模板定制。主要代码如下：

```groovy
// 添加打包用的TASK
def archiveTask = project.task("apk${variant.name.capitalize()}",
                type: ArchiveAllApkTask) {
            theVariant = variant
            theExtension = modifierExtension
            theMarkets = markets
            dependsOn variant.assemble
        }
        def buildTypeName = variant.buildType.name
        if (variant.name != buildTypeName) {
            project.task("apk${buildTypeName.capitalize()}", dependsOn: archiveTask)
        }


// 遍历列表修改APK文件
theMarkets.each { String market ->
            String apkName = buildApkName(theVariant, market)
            File tempFile = new File(tempDir, apkName)
            File finalFile = new File(outputDir, apkName)
            tempFile << originalFile.bytes
            copyTo(originalFile, tempFile)
            PackerNg.Helper.writeMarket(tempFile, market)
            if (PackerNg.Helper.verifyMarket(tempFile, market)) {
                copyTo(tempFile, finalFile)
            } 
        }

```

详细的实现可以查看文件 [PackerNgPlugin.groovy](plugin/src/main/groovy/com/mcxiaoke/packer/ng/PackerNgPlugin.groovy) 和文件 [ArchiveAllApkTask.groovy](plugin/src/main/groovy/com/mcxiaoke/packer/ng/ArchiveAllApkTask.groovy)

### 同类工具

- [**gradle-packer-plugin**](https://github.com/mcxiaoke/gradle-packer-plugin) - 旧版渠道打包工具，完全使用Gradle系统实现，能利用Android提供的productFlavors系统的条件编译功能，无任何兼容性问题，方便集成，但是由于每次都要重新打包，速度比较慢，不适合需要大量打包的情况。（性能：200个渠道包需要一到两小时）
- [**Meituan-MultiChannelTool**](https://github.com/GavinCT/AndroidMultiChannelBuildTool) - 使用美团方案的实现，在APK文件的`META-INF`目里增加渠道文件，打包速度也非常快，但读取时需要遍历APK文件的数据项，比较慢，而且以后可能遇到兼容性问题
- [**MultiChannelPackageTool**](https://github.com/seven456/MultiChannelPackageTool) - 将渠道写入APK文件的注释，这个项目没有提供Gradle插件，只有命令行工具，不方便CI集成，使用ZIP文件注释的思路就是来自此项目


------

## 关于作者

#### 联系方式
* Blog: <http://blog.mcxiaoke.com>
* Github: <https://github.com/mcxiaoke>
* Email: [github@mcxiaoke.com](mailto: github@mcxiaoke.com)

#### 开源项目

* Next公共组件库: <https://github.com/mcxiaoke/Android-Next>
* Gradle渠道打包: <https://github.com/mcxiaoke/gradle-packer-plugin>
* EventBus实现xBus: <https://github.com/mcxiaoke/xBus>
* Rx文档中文翻译: <https://github.com/mcxiaoke/RxDocs>
* MQTT协议中文版: <https://github.com/mcxiaoke/mqtt>
* 蘑菇饭App: <https://github.com/mcxiaoke/minicat>
* 饭否客户端: <https://github.com/mcxiaoke/fanfouapp-opensource>
* Volley镜像: <https://github.com/mcxiaoke/android-volley>

------

## License

    Copyright 2014, 2015, 2016 Xiaoke Zhang

    Licensed under the Apache License, Version 2.0 (the "License");
    you may not use this file except in compliance with the License.
    You may obtain a copy of the License at

       http://www.apache.org/licenses/LICENSE-2.0

    Unless required by applicable law or agreed to in writing, software
    distributed under the License is distributed on an "AS IS" BASIS,
    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
    See the License for the specific language governing permissions and
    limitations under the License.


# Cocos2dx-lua支持x86_64解决方案


前不久收到了 Google Play 的警告邮件，要求上架的app需要对每个原生 32 位架构，都必须同时提供相应的 64 位架构。

例如：
* 对于 ARM 架构，有 armeabi-v7a (32 位) 就必须有 arm64-v8a (64 位)。
* 对于 x86 架构，有 x86 (32 位) 就必须有 x86_64 (64 位)。

本文详细介绍了对于 Cocos 版本比较老的游戏项目，如何能够支持 x86_64 的解决方案与踩坑。希望对有同样需求的伙伴有所帮助。

## 准备工作
首先利用官方给出的方法[ 使用 APK 分析器查找原生库 ](https://developer.android.google.cn/distribute/best-practices/develop/64-bit?hl=zh-cn#apk-analyzer)，可以看到项目所支持的架构以及用到的so文件。如果你当前的应用没有支持 x86 就不需要额外再做对 x86_64 的支持了。

接下来对照上面所有缺少的so文件分别寻找其 x86_64 版本。可分为三类：

1. 项目自身依赖的库。
2. 接入第三方 SDK 时依赖的库。如 Bugly ，极光推送等。
3. Cocos 依赖的第三方库。

由于我们自己开发的 SDK 已经支持了 x86_64 ，所以下面分别说明后两种 so 文件的解决方案。 

### 第三方 SDK
对于游戏接入的第三方 SDK ，如果是比较新的 SDK ，一般官方会直接提供 x86_64 的 so 文件；而对于那些比较老的库，可以尝试根据当前的版本号来下载旧版本。有的官网不提供旧版本的下载链接，可以去 github 上通过历史提交记录查找。找到对应版本的 SDK 后，如果没有提供 x86_64 的动态库，需要下载源文件，重新编译。

Tips: 下载现成的动态库时，可以把其他已有架构的 so 文件一并下载，然后通过 Beyond Compare 的十六进制比较功能，与项目中已有的 so 文件进行对比，确保动态库版本的一致性。

### Cocos 动态库
这里我根据 Android.mk 文件，使用[ ProcessOn ](https://www.processon.com)提供的思维导图功能，针对我们的项目整理了 cocos2dx_lua_static 的依赖树：

![Cocos 第三方库依赖关系](/images/20200111/cocos2d_lua_static.png)

其中绿色的部分只依赖了静态库，蓝色则是我们需要关注的 so 文件。可以看到，Cocos 有很多第三方库，如果一个一个寻找需要花费大量的时间。Cocos团队虽然明确说了不提供对 x86_64 的支持，但是帮助我们整理了所有第三方库的对应版本和源文件，方便我们重新编译。

首先通过cocos目录下的`external/version.json`文件，可以看到当前各个库的版本。利用此文件从这个[ github 链接 ](https://github.com/cocos2d/cocos2d-x-3rd-party-libs-src)中找到相匹配的版本并 clone 源代码。

然后运行脚本：
```shell
./build.sh -p=android --libs=all --arch=x86_64 --mode=release
```

将生成的 so 文件放到项目中对应的 x86_64 文件夹中即可。

## 配置 & 编译
由于历史原因，我们项目采用的是先用 ndk-build 对 cocos 相关的库和 wwise 库进行编译，然后再通过 gradle 编译。首先需要在配置文件中添加 x86_64 :

### ndk-build 配置修改
```mk
# Application.mk
APP_ABI := armeabi-v7a arm64-v8a x86 x86_64
```

### gradle 配置修改
```gradle
// build.gradle
android {
    defaultConfig {
        ndk {
            abiFilters 'armeabi-v7a', 'arm64-v8a', 'x86', 'x86_64'
        }
    }
}
```

如果使用了bugly，记得在生成和上传符号表时添加 x86_64 。

### 代码兼容
在编译阶段还要注意的一点是对 32 位代码移植 64 位的处理，比如在编译 cocos 第三方库 curl 时需要对数据类型做兼容。
```c++
#if defined(__aarch64__) || defined(__x86_64__)
#define CURL_SIZEOF_LONG            8
#define CURL_TYPEOF_CURL_OFF_T      long
#define CURL_FORMAT_CURL_OFF_T      "ld"
#define CURL_FORMAT_CURL_OFF_TU     "lu"
#define CURL_FORMAT_OFF_T           "%ld"
#define CURL_SIZEOF_CURL_OFF_T      8
#define CURL_SUFFIX_CURL_OFF_T      L
#define CURL_SUFFIX_CURL_OFF_TU     UL
```
如果这些在去年升级 arm64 的时候已经做过了，这一步会比较轻松。

## 开始测试
### 测试方法
执行完上述的几个步骤后，就可以开始测试了。目前市面上还没有 64 位的 x86 安卓设备，只能从 Android Studio 上配置 x86_64 的安卓模拟器。安装 apk 的时候需要配合`--abi`参数使用 adb 安装，指定使用我们新加的 x86_64 ，或者也可以直接出一个只带有 x86_64 库的包，确保应用运行时用的不是老的 x86 库。
### 踩坑
在第一次测试时，我们的游戏一启动就会闪退。通过调查报错日志，发现当前版本的 luagit 不支持 64 位的 x86 架构，导致创建 lua state 失败。于是从 luagit 官网下载了最低支持的版本 2.0.5 ，重新编译、打包、安装后就可以正常进入游戏了。

## 后续
Google 并不要求支持所有的 64 位架构，但是对于已经支持的每种原生 32 位架构，就必须包含对应的 64 位架构。

在实际开发的过程中，我们一些 SDK 的 x86_64 来不及测试，也很难测试（没有 x86_64 的设备），无法保证其稳定性。因为 pc 上大部分的 x86 模拟器可以通过翻译 ARM 来正常运行游戏，所以最终我们决定去掉对 x86 的支持来满足谷歌的要求。目前线上的各种手机跑起来没有发现问题，同时也减小了整个安装包的体积。



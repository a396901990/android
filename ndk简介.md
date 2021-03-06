##NDK简介

NDK（Native Development Kit）是一套工具集，允许你在Android应用中嵌入c或c++。  

使用NDK的好处主要有以下4点：

- 安全：由于apk的java层代码很容易被反编译，而C/C++库反汇难度较大。

- 重用：可以方便地使用现存的开源库。大部分现存的开源库都是用C/C++代码编写的。

- 效率：将要求高性能的应用逻辑使用C开发，从而提高应用程序的执行效率。

- 移植：用C/C++写得库可以方便在其他的嵌入式平台上再次使用。


##NDK关键词
###ndk-build:
这个shell构建脚本是NDK的核心。
  
- 它决定需要去构建什么
- 生成二进制文件
- 拷贝二进制文件到项目目录

###Native shared libraries:
本地共享库，本地代码经过编译后生成的二进制文件`.so``.dll`

###Native static libraries:
本地静态库，用来连接其他代码库

###Java Native Interface (JNI):
Java平台的重要特性，允许Java代码和其他本地语言进行交互，如c/c++

###Application Binary Interface (ABI)::
定义了二进制文件（尤其是.so文件）如何运行在相应的系统平台上。在Android系统上，每一种CPU架构对应一个ABI：  
armeabi（默认），armeabi-v7a，x86，mips，arm64- v8a，mips64，x86_64。

###Android.mk:
必备组件，在jni文件夹下，用来描述NDK构建系统。**ndk-build**脚本从该文件中读取定义的模块，名字和需要编译的资源文件等。

###Application.mk:
可选构建文件，和Android.mk一样，放在jni目录下，这个文件列举和描述了你应用模块所需的东西，如指定的ABIs编译平台，工具链等。

##Eclipse中NDK使用流程：
- 创建Android项目，创建java文件，并声明本地方法

- 编译程序，确保生成了`.class`文件
- 使用**javah**命令，生成jni目录和`.h`头文件
- 在jni目录下创建c/c++文件，引入头文件，编写本地方法的实现
- 创建**Android.mk**，用来描述你要在jni目录下生成的本地库
- 创建**Application.mk**（可选）来配置目标ABIs, toolchain, release/debug mode, STL等
- 使用**ndk-build**脚本将c/c++文件编译成本地库`.so`
- 打包，运行程序


##Android Studio中NDK使用流程（旧）：
使用Android Studio可以精简上面的流程，上流程：

####创建java文件，并声明本地方法

```
public class JNIHelper {
    public static native String getStringFromNative();
}
```

####然后build一下，得到.class文件。
生成地址：`YourApplication\app\build\intermediates\classes\debug`

####根据生成的class文件，利用javah 生成对应的 .h头文件。
打开termianl执行：
``javah -d jni -classpath /Users/DeanGuo/TestNDK/app/build/intermediates/classes/debug com.dean.testndk. JNIHelper``  

成功后会生成一个jni目录，目录下会有一个`com_dean_testndk_JNIHelper.h`头文件：

```
/* DO NOT EDIT THIS FILE - it is machine generated */
#include <jni.h>
/* Header for class com_dean_testndk_JNIHelper */

#ifndef _Included_com_dean_testndk_JNIHelper
#define _Included_com_dean_testndk_JNIHelper
#ifdef __cplusplus
extern "C" {
#endif
/*
 * Class:     com_dean_testndk_JNIHelper
 * Method:    getStringFromNative
 * Signature: ()Ljava/lang/String;
 */
JNIEXPORT jstring JNICALL Java_com_dean_testndk_JNIHelper_getStringFromNative
  (JNIEnv *, jclass);

#ifdef __cplusplus
}
#endif
#endif
```
####编写c/c++文件，来实现头文件中的方法。
```
#include "com_dean_testndk_JNIHelper.h"

JNIEXPORT jstring JNICALL Java_com_dean_testndk_JNIHelper_getStringFromNative
  (JNIEnv * env, jobject jclass) {
    return (*env)->NewStringUTF(env, "return from native c");
  }
```

####编译生成.so文件。
传统方法是使用Android.mk，Application.mk进行配置，最后通过ndk-build脚本来生产.so本地库。
使用Android Studio可以将配置和执行脚本的工作都交给Gradle。

打开工程的build.gradle配置文件，在defaultConfig中添加`ndk`标签用来配置.so的名字和abi架构，如下：

```
    defaultConfig {
        applicationId "com.dean.testndk"
        minSdkVersion 14
        targetSdkVersion 23
        versionCode 1
        versionName "1.0"

        ndk{
            moduleName "test"			//生成的so名字
            abiFilters "armeabi", "armeabi-v7a", "x86"	//指定abi体系结构下的so库，不指定使用默认方案
            //  stl "stlport_shared"
            //  ldLibs "log", "z", "m"
            //  cFlags "-I/some/include/path"  
        }
    }
```
>目前需要在**gradle.properties**中添加`android.useDeprecatedNdk=true`来支持这种ndk标签格式（因为出了新的ndk标签方式，下一篇会讲到）

生成好的so文件可以在`app/build/intermediates/ndk/debug/lib`中看到。

####编写代码，打包执行
本地库已经生成，现在就通过代码来加载使用了：

```
public class MainActivity extends AppCompatActivity {

    static {
        System.loadLibrary("test");
    }

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        Log.i("ndk_log : ", JNIHelper.getStringFromNative());
    }
}
```
> 执行后控制台打印：I/ndk_log :: return from native c

##Android Studio 2.2中NDK的使用（新）：
在最新的Android Studio2.2的preview版中，增加全新的ndk支持，使用了新的gradle，以及DSL语言。

新的NDK需要使用新的Gradle插件和新的Android插件来支持！
###gradle-experimental plugin
修改项目（project）的buidle.gradle文件，使用全新的gradle插件：

```
buildscript {
    repositories {
        jcenter()
    }
    dependencies {
        classpath 'com.android.tools.build:gradle-experimental:0.7.0'
    }
}
```
> 需要gradle-2.10以上的支持

###com.android.model.application/library
由于全新的**gradle-experimental**插件使用了新的DSL语言，所以也需要用新的android插件``com.android.model.application``／``com.android.model.library``来替换老版中的``com.android.application``／``com.android.library plugins``:

####老版本DSL：

```
apply plugin: 'com.android.application'

android {
    compileSdkVersion 23
    buildToolsVersion "23.0.2"

    defaultConfig {
        applicationId "com.dean.testndk"
        minSdkVersion 14
        targetSdkVersion 23
        versionCode 1
        versionName "1.0"


        ndk{
            moduleName "test"			//生成的so名字
            abiFilters "armeabi", "armeabi-v7a", "x86"	//指定输出abi体系结构下的so库
            //  stl "stlport_shared"
            //  ldLibs "log", "z", "m"
            //  cFlags "-I/some/include/path"
        }
    }


    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android.txt'), 'proguard-rules.pro'
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile 'com.android.support:appcompat-v7:23.3.0'
}
```

####新版DSL：
```
apply plugin: 'com.android.model.application'

model {
    android {
        compileSdkVersion = 23
        buildToolsVersion = "23.0.2"

        defaultConfig {
            applicationId = "com.dean.testndk"
            minSdkVersion.apiLevel = 14
            targetSdkVersion.apiLevel = 23
            versionCode = 1
            versionName = '1.0'
        }
        ndk {
            moduleName = 'test'
            ldLibs.addAll(['android', 'log'])
        }
        buildTypes {
            release {
                minifyEnabled = false
                proguardFiles.add(file('proguard-android.txt'))
            }
        }
    }
}

dependencies {
    compile fileTree(dir: 'libs', include: ['*.jar'])
    testCompile 'junit:junit:4.12'
    compile 'com.android.support:appcompat-v7:23.3.0'
}
```
简单对比新老DSL还是有很多变化的。而目前这个版本的gradle已经明确说是**experimental**的，所以还是先等正式版出来为好。
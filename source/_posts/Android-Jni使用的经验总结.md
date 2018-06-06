---
title: Android Jni使用经验总结
date: 2018-01-20 16:07:46
tags:
---

---
这段时间因为开始编写与硬件结合的一个软件项目，使用到了大量的jni调用，这里总结一下使用要点  

<!--more-->

首先开始我们的环境搭建，Android studio的环境搭建还是比较简单的，在Sdk Manager里找到NDK并安装，安装好的NDk一般位于你的sdk文件夹下的ndk-bundle，然后将该路径配置到你系统变量的path里面去  
cmd命令输入  
``` bash
ndk build
```
如果未提示"ndk-build不是系统命令"就表示NDK环境配置完成了!

对于app来说，一些对图片的处理、视频的处理等等需要复杂算法支持的功能，很多都会选择用C语言写成，然后APP端去调用C语言方法，得到经过算法处理之后的结果，jni就是Java和C之间调用的桥梁  
在前端时间的项目中，因为需要大量的使用类似活体检测、人脸校准之类的算法，所以必不可少的接触到了jni的使用

#### Android studio2.2.3之前的jni调用（此处举例）  

##### 在Java的demo里编写如下代码
``` java
public class MainActivity extends Activity {  
  
    static {  
        System.loadLibrary("JniTest");  
    }  
  
    public native String getStringFromNative();  
  
    @Override  
    protected void onCreate(Bundle savedInstanceState) {  
        super.onCreate(savedInstanceState);  
         
        getStringFromNative(); 
    }  
}  
```
此处根据你写的native的方法名等下生成C语言的.h头文件

##### 在app文件夹下的build.gradle文件里面配置Ndk相关信息  
``` java
ndk {  
    moduleName "JniTest"  
    ldLibs "log", "z", "m"  
    abiFilters "armeabi", "armeabi-v7a", "x86"  
}
```
基本看代码可以看懂，指定加载so库的名称为JniTest，这个只是为了举例取的名称，加载的abi平台为armeabi、armeabi-v7a和x86
``` java
sourceSets{  
    main{  
        jniLibs.srcDirs = ['libs']  
    }  
}  
```
指定加载so库的路径为libs  

##### 生成需要的.h头文件
在Android Studio下方的Terminal命令行里面执行生成命令,如果没有Terminal,到"View->Tool Windows->Terminal"里面打开该面板
操作命令:  
``` bash
javah -d jni -classpath <SDK_android.jar>;<dir path> 加载So所在的class包名+class名  
```
后面的 “dir path” 加载So所在的class包名+class名 是相对于你当前所在目录的路径;

参考命令:
``` bash
C:\git_source\demo\JniTest>javah -d jni -classpath C:\studio\android-sdk-windows\platforms\android-23\android.jar;app\src\main\java com.example.jni.jnitest.MainActivity  
```
这里的后面一部分是相对于"C:\git_source\demo\JniTest"目录的路径
##### 注意：很多人写命令行的时候容易写错，`app\src\main\java`后面是空格 接 包名.类名

命令执行成功会自动生成一个jni文件夹和对应的"包名.h"文件,这里的文件名是:com_example_jni_jnitest_MainActivity.h  
<figure>    
   <img src="http://img.blog.csdn.net/20160803150457429?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast" >
</figure>  


.h文件中的代码如下
``` c++
JNIEXPORT jstring JNICALL Java_com_example_jni_jnitest_MainActivity_getStringFromNative(JNIEnv * env, jobject jObj){  
      return (*env)->NewStringUTF(env,"Hello From JNI!");    
  }  
```
这里的方法就是根据你的native方法生成的头信息,我们来解析一下意思
`JNIEXPORT和JNICALL`：jni关键字就跟Java里的关键字一样
`jstring`：   方法的返回值为string
`Java_com_example_jni_jnitest_MainActivity_getStringFromNative` ：  格式为Java\_+包名\_+调用jni方法的类名\_+方法名  如果包名里包含下划线，则用_1代替

接下来就是编写你的C语言代码了，如果你不会写C语言代码，那就是得有底层给你.cpp的文件或者.C的文件，我们只需要将我们刚刚生成的头文件里的方法声明，去替换C语言文件里的方法声明，这样C语言的方法就可以被Java调用了

##### 编辑Android.mk和Application.mk  
按照前面第一步的图里面的jni包里面有一个Android.mk文件和Application.mk文件,这里我们需要创建这两个文件，还有一个main.c文件就是我们需要生成so库的文件，Android.mk文件和Application.mk文件是用来控制编译So文件的, Android.mk文件控制So文件如何编译, Application.mk文件控制支持的架构平台.

Android.mk文件:
``` java
LOCAL_PATH := $(call my-dir)  
  
include $(CLEAR_VARS)  
#bzlib模块  
bzlib_files := \  
    main.c  
  
LOCAL_MODULE := libbz  
LOCAL_SRC_FILES := $(bzlib_files)  
include $(BUILD_STATIC_LIBRARY)  
  
#bspath模块  
include $(CLEAR_VARS)  
LOCAL_MODULE    := main  
LOCAL_SRC_FILES := main.c  
LOCAL_STATIC_LIBRARIES := libbz #引入libbz库  
  
include $(BUILD_STATIC_LIBRARY)  
  
include $(CLEAR_VARS)  
  
LOCAL_MODULE    := JniTest  
LOCAL_SRC_FILES := main.c  
LOCAL_STATIC_LIBRARIES := main  
LOCAL_LDLIBS := -llog#加入log  
  
include $(BUILD_SHARED_LIBRARY)  
```
`"bzlib_files :"`里面放的是所有的c源文件;`"LOCAL_MODULE := JniTest"`表示生成的So库名,因为我加载So库的名字是"JniTest"所以这里需要同名;
``` java
static {  
        System.loadLibrary("JniTest");  
    }  
```

Application.mk文件:
``` java
APP_CFLAGS += -Wno-error=format-security  
APP_ABI := armeabi  
```

APP_ABI := armeabi 表示只支持armeabi架构的Jni,如需添加其他架构可在后面追加

##### 最后一步：生成.so文件
在CMD命令行里面cd到项目的目录下, 输入"ndk-build"生成So包
![](http://img.blog.csdn.net/20160803160351099?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
命令执行成功后在工程的libs文件夹下面生成一个So文件,如下:
![](http://img.blog.csdn.net/20160803163127532?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
如果不添加Application.mk文件,会生成所有架构的So文件,如下:
![](http://img.blog.csdn.net/20160803163555931?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

编译出来的So默认是放在工程里面的libs文件夹下的,需要将它移到app文件夹下的libs里面!

#### 2.2.3之后的Jni调用  
从AndroidStudio2.2开始，studio可以利用NDK直接编译C/C++代码。AndroidStudio 支持Cmake和NDK-BUILD 工具编译本地代码，但是默认方式为Cmake。
在用AndroidStudio开发native应用之前先要下载好NDK，Cmake，LLDB(本地代码调试工具)，可以直接通过studio的SDK Manager进行安装，安装完成如下路所示： 
![](http://img.blog.csdn.net/20161222172524036?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDY1NzIxOQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

创建可以编译C/C++的工程
![](http://img.blog.csdn.net/20161222172604318?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDY1NzIxOQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)
直接选择next，其他跟普通工程无异，直到Customize C++ support 界面，如下图所示： 
![](http://img.blog.csdn.net/20161222172650742?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDY1NzIxOQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

1. C++ Standard：Toolchain Default 会使用默认的 CMake 设置。
2. Exceptions Support：对 C++ 异常处理的支持,如果启用此复选框，Android Studio 会将 -fexceptions 标志添加到模块级 build.gradle 文件的 cppFlags 中，Gradle 会将其传递到 CMake。
3. Runtime Type Information Support：如果希望支持 RTTI，选中此复选框,Android Studio 会将 -frtti 标志添加到模块级 build.gradle 文件的 cppFlags 中，Gradle 会将其传递到 CMake。

选择好后，最后点击Finish 
稍等片刻，简直不敢相信自己的眼睛，简直辣眼睛，自动生成项目结构如下如所示： 
![](http://img.blog.csdn.net/20161222172721226?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvdTAxMDY1NzIxOQ==/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

所有的模板已经生成好了，只需要往里面填代码即可！无需生成.so库，直接可以调用jni方法，studio编译会自动生成.so库，大大简化了以前要自己生成.so库的过程！






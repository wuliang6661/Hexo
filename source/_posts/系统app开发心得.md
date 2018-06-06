---
title: 系统Lanucher app开发
date: 2018-01-20 11:23:33
tags:
---

---

这几天因为公司开发的一个门禁一体机，需要把app作为开机自启动的程序，成为一个系统lanucher，下面记录一下在开发过程中遇到的坑  
<!--more-->
## 怎么获取系统app的权限  
程序刚开始写的时候，就遇到各种问题，比如  
1.程序可以设置Android的系统时间  
2.需要可以使系统恢复出厂设置  
等等巴拉巴拉一堆需要获取系统权限的东西，于是遍寻方案，用了一下方法解决  
1.在AndroidManifest.xml的根节点需要加入下面一句  

``` java
android:sharedUserId="android.uid.system"
```   
这个时候就可以声明一些敏感的系统权限啦，比如  
``` java
<uses-permission android:name="android.permission.SET_TIME" />
```   

接下来一个app想要成为系统的app就只差一步了，获取系统的签名进行打包，获取系统签名你需要以下几个文件  
目标Android系统的“build/target/product/security”目录获取以下两个文件

``` bash
platform.pk8
```  

``` bash
platform.x509.pem
```  
以及签名工具
``` bash
signapk.jar
```  
使用Android studio的build apk打出无签名的apk包，改名为Demo.apk，放入存放这三个签名文件的文件夹中，cmd命令执行下面这句话

``` bash
java -jar signapk.jar platform.x509.pem platform.pk8 Demo.apk signedDemo.apk
``` 
你就会发现你需要的系统apk  signedDemo.apk  就编译成功了，这个apk就是成功集系统权限于一体的包啦&#33;

## apk打入系统，开机自启动  
接下来就是遇到的第二个坑了，当我把apk打成系统包之后，怎么把他打入系统，开机自启动呢  
于是又开始了百度之旅，找呀找呀找答案，果然万能的百度给了我答案，只要将apk push进系统的system/app目录，就会变成预加载的系统程序了，吼吼，然后开始实验  
可是还没开始就遇到一个问题，发现system/app目录是只读的，于是需要执行下面这句话给与我们写入权限  

配置好adb的运行环境之后，开始依次执行下面的命令  
``` bash
adb shell
``` 
``` bash
su
``` 
``` bash
mount -rw -o remount /dev/block/platform/ff0f0000.rksdmmc/by-name/system /system/
``` 
这样我们就获取到system目录的读写权限啦  
可是当我天真的以为这样就可以把apk push进去的时候，发现我只是获得了读写权限，却没有root权限，依然失败，于是苦思冥想有了思路，先把apk push进
任意一个有root权限文件夹，然后copy到system/app下不就可以绕过去了，于是又开始敲命令行了  

这里我有一个临时的手机文件夹存放资源的  /data/rk_backup/  当然你们也可以另外放在data底下,开始执行以下命令
``` bash
adb push signedDemo.apk /data/rk_backup/
``` 
进入shell
``` bash
adb shell
``` 
``` bash
su
``` 
在system/app/下创建一个文件夹用来盛装apk
``` bash
mkdir /system/app/Lanucher
```  
拷贝apk到system/app/下
``` bash
cp /data/rk_backup/signedDemo.apk /system/app/Lanucher/
```   
查看是否存在apk
``` bash
cd /system/app/Lanucher
```   
``` bash
ls
```   
如果apk已经在目录底下，这个时候就说明我们成功啦，接下来赋予apk权限啦 执行
``` bash
chmod 644 signedDemo.apk
```  
``` bash
cd ..
```  
为盛装apk的文件夹赋予权限
``` bash
chmod 755 Lanucher
```  
这样我们的apk就正式打入系统了，如果我们想让手机开机就启动此apk的话呢，只需要删除system/app/目录里的手机桌面程序就行了，言归正传，这时候就只剩最后一步了，重启手机，是不是很激动，见证奇迹的时候到了  
当我们重新开机，就会发现，你的程序已经默认安装到了手机里了，你可以随意的更改系统时间以前不能做到的操作了  











---
layout: post
category : React-Native
title: 'React-Native安装笔记'
tagline: ""
tags : [React-Native]
---

* auto-gen TOC:
{:toc}

## React-Native安装

React Native中文网的安装说明文档:[搭建开发环境](http://reactnative.cn/docs/0.31/getting-started.html#content)  

1、按照上面的安装文档,所有软件的安装过程都比较顺利 
  
2、IOS的环境搭建好后,切换到项目根目录,直接执行react-native run-ios,成功拉起IOS版的demo界面
  
3、Android踩了个坑,根据文档安装完所有步骤后，执行react-native run-android，这时报了个错误：
    
    :app:assembleDebug UP-TO-DATE
    :app:installDebug FAILED
    
    FAILURE: Build failed with an exception.
    
    * What went wrong:
    Execution failed for task ':app:installDebug'.
    > com.android.builder.testing.api.DeviceException: No connected devices!
    
    * Try:
    Run with --stacktrace option to get the stack trace. Run with --info or --debug option to get more log output.
    
    BUILD FAILED
    
    Total time: 10.306 secs
    Could not install the app on the device, read the error above for details.
    Make sure you have an Android emulator running or a device connected and have
    set up your Android development environment:
    https://facebook.github.io/react-native/docs/android-setup.html

看了一下报错内容,原来是要先启动emulator,于是执行emulator的启动命令:  

    emulator：emulator @`emulator -list-avds` //适用于当前只有一个avd

这时又报了一个错误:

    emulator: ERROR: x86 emulation currently requires hardware acceleration!
    Please ensure Intel HAXM is properly installed and usable.
    CPU acceleration status: HAX is not installed on this machine (/dev/HAX is missing).

看报错信息说是HAX没有装,于是打开Android Studio的SDK Manager面板查看到Intel x86 Emulator Accelerator (HAXM Installer)已经是installed的状态  

在网上查了一下资料,原来SDK Manager只是把文件下载下来了，HAX其实是没安装的

可以进入到Android SDK的HAX目录下,执行silent_install.sh静默安装:

    cd /Users/yuyongjia/Library/Android/sdk/extras/intel/Hardware_Accelerated_Execution_Manager/
    sudo ./silent_install.sh

安装完毕后,再重新执行,成功拉起Demo界面:

    react-native run-android
    

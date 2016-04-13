---
title: 利用XCTool实现自动化打包
date: 2016-02-13 13:17:21
categories: iOS
tags: [XCTOOL, SHELL, LINUX命令]
---
当你在认真做你自己的事的时候，突然测试来说帮我打个包吧，虽然心里面可能有千万的不愿意，但是你还得老老实实的插上数据线，连接上手机编译程序打个包给他，作为一个偷懒的程序员你能容忍做这种完全没有意义的事？此篇主要介绍如何利用shell脚本实现自动化打包，这样我们就可以帮节约的时间留下来去勾搭妹纸了。

<!--more-->



# XCTool

xctool是Facebook开源的一个命令行工具，用来代替苹果的xcodebuild工具。

功能如下：

* 像xcode一样能够编译程序
* 编译的结果能够以结构化的形式输出
* 编译内容输出彩色且方便阅读


## 安装XCTool

最简单的办法就是用homebrew安装xctool，这边就不介绍如何安装homebrew了。

` brew install xctool`

注：在用brew安装xctool的时候可能会出现can not install with HEAD error，当时尝试很多办法都没有搞定，像卸了重装... 终极解决方案：直接删除/usr/local里面里面所有的文件，*这边会导致你之前安装的一些插件会被删除，像CocoaPods*。

## 编译

`xctool -workspace ${project_path}/${project_name}.xcworkspace -scheme ${project_name} archive -archivePath ${project_path}/build/Debug-iphoneos/${project_name}_Debug.xcarchive -configuration ${build_Mode} || exit
`
 
# XcodeBuild

shell里面会利用xcodebuild进行打包、清理工程

## 打包
`xcodebuild -exportArchive -archivePath ${project_path}/build/Debug-iphoneos/${project_name}_Debug.xcarchive -exportPath ${buildExportPath}/debug.ipa -exportFormat ipa \
-exportProvisioningProfile "${profile_name}" || exit`

## 清理工程
`xcodebuild clean -configuration ${build_Mode} || exit`


# Linux命令
shell里面会涉及到一些Linux命令：

`echo "输出内容"`

`echo '输出内容'`

输出你想要输出的东西，*‘ ’表示要输出的内容为纯字符串，不可引用变量，“”表示输出的内容可包含变量*

`cd ..`

回退上一级目录

`makdir`

新建一个目录

`rm -rf "content"`


删除文件夹里面的所有内容

**以上内容纯属虚构，若不感兴趣，完全可以跳过，直奔使用~**


# 使用教程


1. 将ipa_build这个文件夹移到和.xcodeproj或.xcworkspace同一级目录

2. 打开build_debug或者build_release文件，替换profile_name字段，替换为你要打包项目的profile name

3. 打开你的字段  `cd 脚本所在的目录` -> `chmod 777 build_debug.sh (仅第一次的时候需要)` -> `./build_debug.sh`

4. 查看你的桌面即可以查看导出的IPA包

<p>
> 感谢大家花费时间来查看这篇blog，需要下载shell的同学请猛戳[Git](https://github.com/PanXianyue/ipa_build)。
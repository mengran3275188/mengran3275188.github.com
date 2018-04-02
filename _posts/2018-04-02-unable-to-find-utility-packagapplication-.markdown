---
layout:      post
title:       Solution to "unable to find utility packageapplication"
data:        2018-04-02
tags:
    - Solution
---

# xrun: error: unable to find utility "PackageApplication", not a developer tool or in PATH

项目里使用了xcode工程的自动构建，用命令xcrun从*.app生成*.ipa。Xcode升级到8.3后，自动构建时会报错，提示下面的这个错误：

> xcrun:error:unable to find utility "PackageApplication", not a developer tool or in PATH

通过对比新旧版本的xcode工程，发现新版的Xcode移除了PackageApplication。最简单暴力的方法就是直接从旧版本的xCode里拷贝一份过来，放到新版Xcode的工具目录,目录如下：
> /Application/Xcode.app/Contetns/Developer/Platforms/iPhonesOS.platform/Developer/usr/bin

然后增加可执行权限：

> chmode +x /Application/Xcode.app/Contetns/Developer/Platforms/iPhonesOS.platform/Developer/usr/bin/PackageApplication

在这里存一下[PackageApplication](/packages/PackageApplication.zip) 方便下载





---
title: Avalonia构建跨平台项目中安卓项目报错的处理
date: 2025-04-20 19:19:31
tags:
- 笔记
- avalonia
categories:
- [笔记,.NET,Avalonia]
---

## 第一步
按照[Install .NET for Android dependencies](https://learn.microsoft.com/en-us/dotnet/android/getting-started/installation/dependencies)安装Android-SDK
然后安装JDK，去java官网安装，同时目前avalonia要求的jdk版本是17，注意不要弄错了，虽然可能未来会变吧，当你看到这篇文章的时候

## 第二步
配置项目
将安卓子项目的csproj文件用文本编辑软件打开，在
``` xml
<PropertyGroup>
</PropertyGroup>
```
之中加入
``` xml
<AndroidSdkDirectory>你的SDK路径</AndroidSdkDirectory>
<JavaSdkDirectory>你的JDK路径</JavaSdkDirectory>
```
就可以了
注意不要直接复制过去就不管了，记得把“你的XX路径”改成你电脑上这些东西的安装路径
OK
然后就不会报错了
NICE！
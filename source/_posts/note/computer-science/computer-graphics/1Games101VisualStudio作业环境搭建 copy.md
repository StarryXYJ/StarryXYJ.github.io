---
title: Games101VisualStudio作业环境搭建
date: 2024-04-19 11:51:23
tags:
- 笔记
- 计算机图形学
categories:
- [笔记，计算机科学，计算机图形学]
---

虚拟机的方式虽然环境搭建好了
但是网盘要下载很久，虚拟机也相对比较卡顿
需要更多的硬盘空间

## 下载
下载Visual Studio Installer
选择C++桌面开发
下载完打开Visual Studio
新建项目，把作业压缩包的文件直接拖到项目里

## 配置包环境
Eigen可以直接在nuget下载
下载完要把源代码里所有eigen3删去
至于opencv，我直接nuget下载会报错
所以就直接[下载opencv](https://github.com/opencv/opencv/releases/download/4.9.0/opencv-4.9.0-windows.exe)
手动配置
下载好直接安装
安装完回到VS，右键项目点击属性
在 链接器 - 输入 - 附加依赖项 中添加 opencv_world490d.lib
在 VC\+\+目录 - 包含目录 中添加 安装路径下build内include目录的路径 | 如 E:\code\cpp\opencv\opencv\build\include
在 VC\+\+目录 - 库目录 中添加 安装路径下build/x64/vc16/lib | 如 E:\code\cpp\opencv\opencv\build\x64\vc16\lib


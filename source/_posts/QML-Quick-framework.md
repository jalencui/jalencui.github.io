---
title: 精通QML系列 -- Quick 工程 -- 框架蓝图
author: Jalen Cui
comments: true
categories: Qt
date: 2023-04-05 20:31:22
data: 2023-04-01 17:28:32
tags:
---


# 预备知识
Windows 支持两种类型的应用程序： GUI 程序和 CUI 程序。本文介绍的对象当然是 GUI 程序。Windows 应用程序必须有一个入口点函数，应用程序运行时，这个函数将被调用。C/C++ 开发人员可以使用以下两种入口点函数：
```cpp
Int WINAPI _tWinMain(
    HINSTANCE hInstanceExe,
    HINSTANCE,
    PTSTR pszCmdLine,
    int nCmdShow);

int _tmain(
    int argc,
    TCHAR *argv[],
    TCHAR *envp[]);
```
实际上，操作系统并不调用我们所写入口点函数。相反，它会调用由 C/C++ 运行库实现并在链接时使用 __-entry:__ 命令行选项设置的一个 C/C++ 运行时启动函数。__这些__ 启动函数的用途简单总结如下：
* 获取指向新进程的完整命令行的一个指针
* 获取指向新进程的环境变量的一个指针
* 初始化 C/C++ 运行库的全局变量
* 初始化 C 运行库内存分配函数（ __malloc__ 和 __calloc__ ）以及其他底层 I/O 例程使用的堆（heap）
* 调用所有全局和静态 C++ 类对象的构造函数  

完成所有这些初始化工作之后，C/C++ 启动函数就会调用应用程序的入口点函数（后文简称 `main`）。一个精心设计的应用程序框架，会在 `main` 入口点做一些有条理的工作：
* 解析命令行参数：判断运行目录，判断唤起方式，及其他业务
* 初始化环境：以共享内存的维护的单实例判断、日志系统初始化、多语言初始化、样式表初始化
* 初始化 UI：主窗口的初始化, 如：QMainWindow（qwidget）, Window（qml）
* 开启事件循环：阻塞型 exec 的执行，但不阻塞 __交互__ 和 __UI__ 。exec 可被其他事件循环暂时挂起，比如：模态对话框，`QEventLoop` 同步业务逻辑

由于涉及知识点较广泛，本文仅从某一侧面梳理 quick 工程应有的模样。

# 框架蓝图： 持续迭代中
![](qtquick_bp.png)
# 附记
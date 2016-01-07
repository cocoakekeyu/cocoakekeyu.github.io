---
layout: post
title: 给VIM安装YouCompleteMe插件
categories: vim
---

VIM已有Vundle插件管理，以为安装比较简单，使用vundle安装的。

文件下载完毕提示没有YCM CORE。上网搜索下需要在插件目录编译，去YouCompleteMe的[github](https://github.com/Valloric/YouCompleteMe)可以看到有安装方法的：

Install development tools and CMake: `sudo apt-get install build-essential cmake`

Compiling YCM with semantic support for C-family languages:
```
cd ~/.vim/bundle/YouCompleteMe
./install.py --clang-completer
```
Compiling YCM without semantic support for C-family languages:
```
cd ~/.vim/bundle/YouCompleteMe
./install.py 
```


编译的时候:

1. 提示“CMake Error: your CXX compiler: "CMAKE_CXX_COMPILER-NOTFOUND" was not found. ”

    最近刚装系统，还没有编译器，去下载了g++，`sudo apt-get install g++-4.8`

2. 提示“CMake Error at /usr/share/cmake-2.8/Modules/FindPackageHandleStandardArgs.cmake:108 (message):
          Could NOT find PythonLibs (missing: PYTHON_LIBRARIES PYTHON_INCLUDE_DIRS)”

    貌似找不的python库，上网查了一下，下载最新的python版本，`sudo apt-get install python-dev`

最后，编译真的很久。

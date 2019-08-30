---
style: summer

title: gdb安装与使用
tags:
  - c++
  - gdb
  - docker
---


在linux上跑c++程序总会遇到各式各样的错误，不想用`printf`来debug的话，赶紧入门gdb吧，你会觉得爽的!现在docker的技术已经很成熟了，自己在笔记本上装个docker，装好vim和相关的插件，装个gdb，就可以开始了。

<!-- more -->


## gdb安装 
1. mac上安装gdb
	* 执行下面的命令：
	    ```bash
	    ruby -e"$(curl -fsSL  https://raw.githubusercontent.com/Homebrew/install/master/install)"
	    ```
	* 更新homebrew软件库并安装gdb。
	    ```bash
	    brew search gdb
	    # 检查gdb是否在homebrew的软件库中
	    brew update
	    # 更新homebrew软件库
	    brew install gdb
	    # 安装gdb
	    ```
	* 修改钥匙串  
    	[修改钥匙串详细方法](http://www.jb51.net/os/MAC/405488.html)
	* 结论
		gdb和macos sierra不兼容。后续版本可以兼容，但是不能用于golang的debug, golang官方推荐用`dlv`，试了一下，输出的debug信息不全，还不如goland。
1. 使用docker环境玩gdb
	* 下面命令生成的docker container可以运行gdb：
    ```bash
    docker run --rm -it --security-opt seccomp=unconfined --name ds schickling/rust
    ```
	* 这个命令生成的docker container是一定可以运行gdb的。但是这些容器都比较低端，没有vim，有的连vi都没有！最好使用目录映射，让docker container和宿主机共享一个目录.这样就可以在本地用高端的编辑器来写代码，在docker中用gdb调试了！所以上述命令可以改为：
    ```bash
    docker run -d -v ~/Desktop/zmx/gdb_share:/gdb_share -it --security-opt seccomp=unconfined --name gdbs schickling/rust
    ```
  	* 但是这么一来docker container中还是没有vim，查代码还是不方便，还是要更新apt-get，然后安装vim。



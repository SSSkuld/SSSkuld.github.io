---
title: Windows+Clion环境搭建
date: 2021-11-19 00:23:57
tags: 
- Linux
- CLion
- WSL
categories: Linux
---
## 背景
最近开始学习网络编程，自己的Windows主机上一直没有搭建相关的开发环境，下载Clion后发现缺少一系列的编译工具链。网上查阅资料后，发现使用Windows的WSL工具，不用安装虚拟机就可以方便的拥有一个Linux环境，并且CLion也可以在这个环境下进行编译。

一通尝试下来发现确实方便，Windows+WSL+CLion的工具环境轻量又好用，应该是以后在家的主要工作环境了。
## 步骤
### 1.开启WSL
按 Win+X, 找到 Windows PowerShell (管理员)，并复制执行命令。

Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Windows-Subsystem-Linux

这个命令会激活WSL功能，之后需要重启电脑。
重启电脑之后，win+R，输入 appwiz.cpl，左上角找到“启动或关闭 Windows 功能”，会看到“适用于Linux的Windows子系统”选项处于选中状态。

其实之前这个命令就相当于手动打开这个选中项。
### 2.安装WSL发行版本
在微软应用商城里搜索Ubuntu，选择一个合适的版本安装即可，这里推荐安装Ubuntu 20.04 LTS。

依次获取、安装，国内网络环境下微软应用商店可能存在些延迟，耐心等一下或者翻墙下载均可。

安装完后，打开Ubuntu即可，按照提示依次设置好用户名和密码即可。

接下来可以进行一些Linux常规的环境配置。

首先进行apt换源，我使用的是阿里云镜像。

<b>
【⚠】首先需要备份一下sources.list文件，很重要不要跳过。

sudo cp /etc/apt/sources.list /etc/apt/sources.list.backup
</b>

之后执行vim /etc/apt/sources.list，替换文件内容即可。

deb http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-security main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-updates main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-proposed main restricted universe multiverse
deb http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse
deb-src http://mirrors.aliyun.com/ubuntu/ focal-backports main restricted universe multiverse

之后更新软件
sudo apt-get update
sudo apt-get upgrade

注意，此时如果对sources.list文件操作不当，可能会触发如下报错：
Malformed line 1 in source list /etc/apt/sources.list (type)
The list of sources could not be read.

此时前往/etc/apt目录下，执行 sudo rm sources.list 删除文件
之后sudo touch sources.list
打开文件，将替换内容复制进去并保存即可。
### 3.CLion编译环境配置
打开CLion项目，会提示编译工具链缺失。

选择环境为WSL，CLion会自动检查对应工具的缺失情况，根据报错，缺少哪个，在Ubuntu的终端中对应下载即可。

至此整个流程结束，整个搭建流程还是很快速的。





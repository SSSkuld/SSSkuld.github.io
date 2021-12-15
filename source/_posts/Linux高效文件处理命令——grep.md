---
title: Linux高效文件处理命令——grep
tags:
  - Linux
  - 终端
date: 2021-12-14 21:23:18
categories: Linux
---

## 前言

### MacOS环境配置GNU版本grep
MacOS环境下自带grep命令，不过是BSD版本，使用`grep --version`命令即可查看版本。

该版本对-r -p等参数不支持，需要安装GNU版本的grep命令才可执行。

直接使用Homebrew进行安装即可。
```
brew install grep
```
网上很多教程说要携带`--with-default-names`参数进行grep命令覆盖，但此参数已经被废弃，无法使用。

题外话，如果Homebrew提示需要更新CLT版本，执行`brew update-reset`即可。

安装完GNU版本grep后，需要更新一下对应的`.bashrc`或者`.zshrc`文件。
```
export PATH="/usr/local/opt/grep/libexec/gnubin:$PATH"
```
添加完成后对应执行`source .bashrc/.zshrc`便成功完成替换，此时再次输入`grep --version`看到输出为GNU版本即为替换成功。

### grep命令

基础命令格式如下
```
grep [OPTION] PATTERNS [FILE]
```
也可以配合管道命令｜一起使用
```
[FILE] | grep [OPTION] PATTERNS
```
`PATTERNS`为要查询的匹配串，支持正则表达式。

`OPTION`为命令参数，可以输入多个，常用的参数如下：

`-B(NUM),  --before-context=NUM`
输出匹配行的前NUM行

`-A(NUM),  --after-context=NUM`
输出匹配行的后NUM行

`-C(NUM),  --context=NUM`
输出匹配行的前后NUM行

`-r`

`grep -r [PATTERN] .`
在当前目录下递归搜索PATTERN，可以再添加--exclude-from=FILE或者--exclude-dir=GLOB进行过滤。

`-v`
相当于not操作，查询不包含PATTERN的内容

`-c` 
输出文件匹配PATTERN的总行数

`-n`
输出文件匹配PATTERN的所在行数

`grep [PATTERN1 \| PATTERN2] [FILE]`
使用\|符号匹配多个PATTERN


### 参考文稿

[macOS 使用 GNU 命令](https://blog.cotes.info/posts/use-gnu-utilities-in-mac/)
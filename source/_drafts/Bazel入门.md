---
title: Bazel入门
tags:
categories:
---

# 前言

Bazel是Google开源的一套构建体系，支持多语言，可扩展性好。

在iOS和C++项目中运用很多，Andorid大部分还是Gradle。

本篇博客将先介绍Bazel的基本概念，再介绍如何将一个iOS工程迁移至Bazel构建，最后扩展至所有语言。

# 从构建系统说起
为什么需要构建系统？

可能对于大多数iOS开发者来说，构建系统这个概念是比较陌生的，因为XCode将其内置的足够便利，大部分情况下，开发人员只需要按下command+B等待编译即可。

但实际上，按下command+B后，背后实际是xcodebuild一系列的操作，从依赖分析开始，到编译、链接，最终打包为ipa并安装到目标设备上，而构建系统就是为了管理这些操作。

对于XCode来说，构建系统是Xcodebuild，对于Android来说，构建系统是Gradle。但是XCode将xcodebuild以GUI的形式暴露出来，而Gradle则是以命令行的形式暴露出来。

通过CLI，xcodebuild -h 可以查看xcodebuild的帮助文档。通过命令archive构建包。

主要通过Build Settings、Build Phases、Targets、Schemes等概念，可以将构建过程分解为多个阶段，每个阶段可以配置不同的参数，从而实现定制化的构建。


# Why Bazel

通过上面的介绍，我们初步了解了xcodebuild的基本概念，虽然XCode通过GUI将其集成的方便，但也暴露了其缺点，实现过于黑盒，不利于维护。

而Bazel恰好解决了这些问题，它输入更加透明，可以通过命令行进行构建，并且可以通过配置文件进行定制化，从而实现更高的可维护性。


# Bazel介绍

## WORKSPACE
首先，Bazel需要一个工作区，如何区分工作区，需要依靠WORKSPACE文件。

WORKSPACE是文件系统上的一个目录，其中包含要构建的软件的源文件，以及指向包含构建输出的目录的符号链接。每个工作区目录都有一个名为 WORKSPACE 的文本文件，该文件可能为空，也可能包含对构建输出所需的外部依赖项的引用。

Bazel 还支持 WORKSPACE.bazel 文件作为 WORKSPACE 文件的别名。如果两个文件都存在，则优先使用 `WORKSPACE.bazel` 。



# iOS工程如何迁移到Bazel



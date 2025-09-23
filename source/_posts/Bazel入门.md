---
title: Bazel入门
tags:
  - Bazel
categories:
  - Bazel
date: 2025-09-24 03:10:47
---


# 前言

Bazel是Google开源的一套构建系统，具备可并行、支持分布式编译缓存、可扩展性好、多语言支持等优点。

目前业界中在iOS工程和C++工程中被广泛运用，本文将从iOS开发者的视角带大家初步了解一下BAZEL。

# 从构建系统说起
为什么需要构建系统？

可能对于大多数iOS开发者来说，构建系统这个概念是比较陌生的，因为XCode将其内置的足够便利，大部分情况下，开发人员只需要按下command+B等待编译即可。

但实际上，按下command+B后，背后实际是xcodebuild一系列的操作，从依赖分析开始，到编译、链接，最终打包为ipa并安装到目标设备上，而构建系统就是为了管理这些操作。

对于XCode来说，其自带构建系统是xcodebuild，对于Android来说，最常用的构建系统是Gradle。

XCode将xcodebuild以GUI的形式进行了集成，主要通过Build Settings、Build Phases、Targets、Scheme等概念，将构建过程分解为多个阶段，每个阶段可以配置不同的参数，从而实现定制化的构建。


# Why Bazel

虽然XCode通过GUI将xcodebuild集成的足够方便快捷，但也有其缺点，实现过于黑盒，构建步骤不清晰且缓存利用效率低，尤其在大型项目中构建速度不容乐观。

目前业界将目标转至Bazel基本出发点也都是为了编译效率。

相比之下，Bazel的优势在于构建过程足够清晰，通过DSL文件描述，在输入文件不变的情况下，能保证在不同环境下具有同样的输出，这就为分布式编译及缓存复用提供了前置条件，有利于编译效率的提升。

# Bazel介绍

## WORKSPACE
首先，Bazel需要一个工作区，也就是工程主目录，依靠`WORKSPACE`文件声明。

`WORKSPACE`文件包含要构建的软件的源文件，以及指向包含构建输出的目录的符号链接。每个工作区目录都有一个名为 `WORKSPACE` 的文本文件，该文件可能为空，也可能包含对构建输出所需的外部依赖项的引用。

在iOS工程概念中，`WORKSPACE`对应描述cocoapods体系下的.xcworkspace工程文件及Podfile外部依赖。

Bazel 还支持 `WORKSPACE.bazel` 文件作为 `WORKSPACE` 文件的别名。如果两个文件都存在，则优先使用 `WORKSPACE.bazel` 。

但在Bazel8及后续规划中，将废弃掉WORKSPACE的概念，使用Bzlmod来代替，在Bazel 6.3及更高版本中，`WORKSPACE` 文件已替换为 `MODULE.bazel`文件。

## BUILD files
BUILD文件用来描述每个软件包（package），对应iOS工程概念中的一个Pod仓库或者一个工程仓库。

BUILD文件可以被命名为`BUILD`或者`BUILD.bazel`，如果两者同时存在，则优先使用`BUILD.bazel`。

BUILD文件使用[Starlark](https://github.com/bazelbuild/starlark/)实现，其内部描述了每个package是如何构建的，内容并不包含任何函数，且Starlark程序中不能执行任何I/O，只用来声明构建规则。这一点特性保证了构建过程的可重复性，在BUILD文件输入不变的情况下，会执行相同的构建过程并产生相同的构建产物。

如果代码依赖或者构建命令有变化，需要修改对应的BUILD文件。

## Action Graph
通过BUILD文件和WORKSPACE文件，便能声明整个项目的构建规则，BAZEL通过解析这些内容生成一张有向无环图（DAG）来描述具体的构建过程。

每一个节点是一个构建动作，每一条边代表构建动作的依赖关系。依赖Action Graph，BAZEL可以方便的实现并行构建以及增量构建。

### Aquery
aquery是BAZEL提供的用于查询Action Graph的工具，通过aquery，我们可以清晰的知道构建过程中的任何细节，以此来优化构建过程。
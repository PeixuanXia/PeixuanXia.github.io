---
title: Eclipse JDT - Abstract Syntax Tree (AST) and the Java Model
tags: JDT
key: elipsejdt
---

源教程链接 [Eclipse JDT - Abstract Syntax Tree (AST) and the Java Model](https://www.vogella.com/tutorials/EclipseJDT/article.html#the-java-model-and-the-java-abstract-syntax-tree)

本文是该教程的翻译，初衷是为了帮助自己理解，分享于此希望能和同样使用JDT的程序员进行交流，下文若有理解不当的地方欢迎斧正。

#### 1. Java model 和 Java Abstract Syntax Tree

JDT提供用于访问和控制Java源代码的API。可以通过以下方式访问Java源代码：

- *Java Model*
- _Abstract Syntax Tree(AST)  

##### 1.1. Java Model

每个Java project (Java项目) 都可以使用一个model (模型) 表示。这个模型是Java项目的一个轻量级容错表示。它虽然不能跟AST一样包含那么多信息但是它可以被快速构建。例如*outline view* (eclipse的大纲视图) 就是使用Java model表示出来的。  

Java model被定义在 `org.eclipse.jdt.core` 插件中。Java model使用树形结构表示。该树形结构可以使用下表描述。

| **Project Element**                           | **Java Model element**   | **Description**                                              |
| --------------------------------------------- | ------------------------ | ------------------------------------------------------------ |
| Java project                                  | IJavaProject             | 包含其他所有对象的Java project                               |
| src folder / bin folder / or external library | IPackageFragmentRoot     | 包含源代码或者二进制文件，可以是一个文件夹或者一个library (zip / jar / file) |
| Each package                                  | IPackageFragment         | 每个package都在IPackageFragmentRoot节点下，sub-packages (子包)并不是package的叶子结点，他们都直接列于IPackageFragmentRoot节点下 |
| Java Source File                              | ICompilationUnit         | 源文件通常位于package节点下                                  |
| Types / Fields / Methods                      | IType / IField / IMethod | Types, fields and methods                                    |

##### 1.2. Abstract Syntax Tree (AST)

AST是Java源代码详细的树型表示。AST定义了API用于修改、创建、浏览和删除源代码。

AST的主包是`org.eclipse.jdt.core.dom` (位于`org.eclipse.jdt.core` 插件中)。






---
title: Eclipse JDT - Abstract Syntax Tree (AST) and the Java Model入门
tags: JDT
key: elipsejdt
---

[TOC]

#### 1. Java model 和 Java Abstract Syntax Tree

JDT提供用于访问和控制Java源代码的API。可以通过以下方式访问Java源代码：

- *Java Model*
- _Abstract Syntax Tree(AST)  

##### 1.1. Java Model

每个Java project (Java项目) 都可以使用一个model (模型) 表示。这个模型是Java项目的一个轻量级容错表示。它虽然不能跟AST一样包含那么多信息但是它可以被快速构建。例如*outline view* (eclipse的大纲视图) 就是使用Java model表示出来的。  

Java model被定义在 `org.eclipse.jdt.core` 插件中。Java model使用树形结构表示。该树形结构可以使用下表描述。

| **Project Element** | **Java Model element** | **Description** |
| ------------------- | ---------------------- | --------------- |
| Java project        |                        |                 |




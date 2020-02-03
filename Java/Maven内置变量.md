---
title: Maven内置变量
tags: java
grammar_cjkRuby: true
---
time: `2020-2-3 `
[TOC]

# Maven内置变量
maven pom文件中,所有内容都定义在`<project></project>`中,通常我们自定义的变量位于`properties`节点中:

```xml
<properties>
	<jetty.version>9.4.10.v20180503</jetty.version>
	<spring.version>5.0.8.RELEASE</spring.version>
	<thymeleaf.version>3.0.10.RELEASE</thymeleaf.version>
	<mybatis.version>3.4.5</mybatis.version>
	<mysql.version>5.1.25</mysql.version>
</properties>
```

Maven中为了方便使用,可以直接使用内置变量,不需要自定义:
- ${basedir} 项目根目录
- ${project.build.directory} 构建目录,缺省值为target
- ${project.build.outputDirectory} 构建过程输出目录,缺省值为target/classes
- ${project.build.finalName} 产出物名称, 缺省为`${project.artifactId}-${project.version}` 
- ${project.xxx} 当前pom文件的任意节点的内容
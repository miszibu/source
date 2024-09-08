---
title: maven_tutorial
date: 2020-04-19 22:48:48
tags: [Maven]
---

记录 Maven 相关的命令

<!--more-->



## Maven: Create Project

> mvn org.apache.maven.plugins:maven-archetype-plugin:3.1.2:generate -DarchetypeArtifactId="maven-archetype-quickstart" -DarchetypeGroupId="org.apache.maven.archetypes" -DarchetypeVersion="1.4"  -DarchetypeCatalog=internal

**Note**: -DarchetypeCatalog=internal，当你创建项目时，中途卡住，可以添加该参数。




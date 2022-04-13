---
title: 用Kotlin写Spring的正确方式
categories:
  - 后端
tags:
  - Kotlin
  - Spring
date: 2022-03-19 17:05:05
---

Kotlin与Java完美互操作，但在很多依赖于JVM特性的框架的使用中，并不是那么平滑完美，Spring就是其中之一。有必要总结一下。

# 用Kotlin写Spring

Spring关于Kotlin的支持在[这个手册](https://docs.spring.io/spring-boot/docs/2.0.x/reference/html/boot-features-kotlin.html)中有所阐述，总结起来有这么几点



# 注意事项

Kotlin写Spring，用起来真开心，但有一些小细节，用起来总是觉得有些别扭。

## var还是val



## 空安全的取舍



## 默认final的问题



## data class的各种限制



## getter、setter、property



# 参考

- [Spring Kotlin Support](https://docs.spring.io/spring-boot/docs/2.0.x/reference/html/boot-features-kotlin.html)
- [Spring Kotlin Deepdive](https://github.com/sdeleuze/spring-kotlin-deepdive)
- [Spring Boot Kotlin Demo](https://github.com/sdeleuze/spring-boot-kotlin-demo)
- https://spring.io/blog/2016/02/15/developing-spring-boot-applications-with-kotlin
- https://www.baeldung.com/kotlin/spring-boot-kotlin


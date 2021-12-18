---
title: Spring-Starter注意事项
tags:
---



官方文档说，加上这个注解处理器后，有助于提升性能：https://docs.spring.io/spring-boot/docs/2.1.3.RELEASE/reference/htmlsingle/#boot-features-custom-starter



为什么starter包里面的东西是空的？

- autoconfigure用作自动配置的集合，它的build.gradle配置中，可全都是optional呀：https://github.com/spring-projects/spring-boot/blob/main/spring-boot-project/spring-boot-autoconfigure/build.gradle
- starter用于引入自动配置所需要的依赖，即，autoconfigure中声明为optional的依赖

---
title: Spring源码剖析 - SpringBoot
categories:
  - Spring
tags:
  - Spring
---

本文我们关注SpringBoot是如何Boot的。一个最简单的SpringBoot应用，可以是这样

```kotlin
@SpringBootApplication
class MyApplication

fun main(args: Array<String>) {
    SpringApplication(MyApplication::class.java).run(*args)
}
```


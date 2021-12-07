---
title: 从Spring看单例
tags:
---



用到双检锁的地方：org.springframework.boot.convert.ApplicationConversionService#getSharedInstance

用恶汉模式的地方：AnnotationAwareOrderComparator

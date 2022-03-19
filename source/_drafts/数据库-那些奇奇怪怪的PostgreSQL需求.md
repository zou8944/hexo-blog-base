---
title: 数据库 - 那些奇奇怪怪的PostgreSQL需求
tags:
---



- jsonb_set的正确用法
- 将表A的字段更新到表B中



## 查询jsonb字段不为空的数据(该字段存储数组)

前置知识：

- 字段可为null
- PG中，任意单个字符串都能存储在jsonb类型的字段中，比如null字符串、123等
- 写得尽量优雅

解决方案

```sql
# 理想方式，但如果出现纯量字符串，jsonb_array_length()会报错
select * from resource where jsonb_typeof(images) = 'array' and jsonb_array_length(images) > 0;
# 折中方式: images <> '[]'能够确保安全的原因是，PG会确保空数组就是[]，而不会出现[   ]这样的情况
select * from resource where jsonb_typeof(images) = 'array' and images <> '[]';
```


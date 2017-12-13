---
title: scala
tags:
 - scala
categories: 经验分享
---

### scala作用域保护
```
private[x] 或 protected[x]
```

x指代某个所属的包、类或单例对象。如果写成private[x],读作"这个成员除了对[…]中的类或[…]中的包中的类及它们的伴生对像可见外，对其它所有类都是private。

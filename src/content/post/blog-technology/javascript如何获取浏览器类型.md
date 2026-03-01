---
title: javascript如何获取浏览器类型
publishDate: 2019-03-08 21:08:53
tags: [javascript]
description: javascript如何获取浏览器类型
---

在javascript中我们经常需要通过浏览器类型的不同使用不同的兼容逻辑，那么如何获取浏览器类型呢？我们可以使用navigator.userAgent，再通过字符串的indexOf方法去判断里面有没有诸如"AppleWebKit"等字符串来判定浏览器类型。

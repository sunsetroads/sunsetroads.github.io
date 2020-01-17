---
layout: post
title: Node.js 运行机制详解
categories: Node
description: some word here
keywords: 异步IO，非阻塞，单线程，事件驱动
---

Node.js 是运行在服务器上的 JavaScript, 具备异步IO、非阻塞、单线程、事件驱动等特性。之前对这些特性的理解只是通过一些博客，最近读了朴灵的《深入浅出 Node.js》，对此有了更清晰的理解。

## 服务器模型变迁
### 


---
layout: post
title: 笔记：深入浅出 Node.js
categories: Node
description: some word here
keywords: 异步 IO，非阻塞，单线程，事件驱动
---

Node.js 是运行在服务器上的 JavaScript，具备单线程、异步 IO、事件驱动等特性。之前对这些特性的理解只是通过一些博客，始终不得要领。最近读了朴灵的《深入浅出 Node.js》，对此有了更清晰明确的认识。

Node.js 运行示意图：

![](/images/node/node_event.jpg)
---
layout: post
title: "初识Async & Await"
date: 2024-02-04 10:05:00.000000000 +09:00
categories: [Swift]
tags: [Swift, Concurrency, Async, Await, Task]
---

### 概览

在 WWDC 2021 中，Swift 迎来了一次非常重要的版本更新`Swift 5.5`。这次更新为 Swift 并发编程带来了很大的改变，通过 `async/await`（[SE-0296](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fapple%2Fswift-evolution%2Fblob%2Fmain%2Fproposals%2F0296-async-await.md)）、Structured concurrency（[SE-0304](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fapple%2Fswift-evolution%2Fblob%2Fmain%2Fproposals%2F0304-structured-concurrency.md)）以及 Actors （[SE-0306](https://link.juejin.cn?target=https%3A%2F%2Fgithub.com%2Fapple%2Fswift-evolution%2Fblob%2Fmain%2Fproposals%2F0306-actors.md)），Swift 让开发者可以在更抽象的层面上思考并发场景的解决方式，同时保障了并发场景下的性能和安全性，避免了使用 GCD 等传统并发模型时可能出现的多线程问题。



Swift中的异步与并发

Swift结构话并发

Async & Await


---
tags: []
parent: 'CPU Inheritance Scheduling'
collections:
    - 调度
version: 14551
libraryID: 1
itemKey: 6KTFFNKZ

---
# CPU Inheritance Scheduling

## Abstract

传统的处理器调度机制僵化，只能支持少量的调度类。

CPU inheritance scheduling：任意线程可以充当其他线程的调度器。

## Introduction

存在额外的调度线程，由调度线程来选择下一个任务。

## Motivation

随着应用程序多样性的增加，操作系统需要支持多个共存的处理器调度策略，以满足各个应用程序的需求，并更有效地利用系统的处理器资源。

## CPU Inheritance Scheduling

任何在给定时刻拥有真实 CPU 的线程都可以将其 CPU 暂时捐赠给另一个特定线程，而不是使用 CPU 本身。

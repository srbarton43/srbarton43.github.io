---
title: Cache Simulator
author: Samuel Barton
date: "2023-11-06"
summary: A simulation of different types of memory caches
description: A comprehensive simulation of four different varieties of computer caches
tags: ["simulation", "software", "c"]
cover:
    image: /projects/cachesimulator/cover.png
    caption: Simulator Performance on a 6+ Million Line Tracefile
    hiddenInList: true
TocOpen: false
ShowToc: true
---

## About

This project simulates four different computer hardware caching methods in software, using the LRU strategy to handle evictions. For a trace file with millions of lines, the program can simulate cache performance in under a second on an ubuntu VM.

## Source Code

[Link to Repository](https://github.com/srbarton43/cache_simulator "git repo")

## Available Cache Types

1. direct-mapped
2. two-way set-associative
3. four-way set associative
4. fully associative

## Verbose Output

{{< figure src="/projects/cachesimulator/directmapped.png" caption="Direct-Mapped Caching in Verbose Mode" >}}

{{< figure src="/projects/cachesimulator/fourway.png" caption="Four-Way Set-Associative Caching in Verbose Mode" >}}

## Comparing All Cache Types

{{< figure src="/projects/cachesimulator/comparison.png" caption="Four-Way Set-Associative Caching in Verbose Mode" >}}


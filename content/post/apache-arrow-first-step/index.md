---
title: First Steps Towards Contributing to Apache Arrow's Golang Implementation
date: 2024-10-12 00:00:00+0000
image:
tags:
  - Open Source
weight: 1
draft: true
---

## Introduction

I've been working as a back-end engineer for the past four years, mainly with relational databases like Postgres and building web applications using Java (Spring) and Golang (Echo). Lately, I've been considering making my first serious contribution to an open-source project. I'm interested in delving deeper into low-level data manipulation, which involves more logic-intensive coding than the typical web application development I'm used to.

### Why Apache Arrow?

While exploring open-source projects, I came across Apache Arrow—a platform that offers high-performance data processing tools. After watching some videos and going through the documentation, I found Arrow's approach to in-memory columnar data formats intriguing. I'm thinking about contributing to the project, specifically to the Golang implementation of Apache Arrow.

## Exploring the Golang vs. Java Implementations

My initial goal was to identify features that might be missing in the Golang implementation compared to the more mature Java version. I thought about learning these features through Java and then helping to implement them in Golang as my first contribution. Although I haven't pinpointed specific gaps yet, I've started to learn more about the Apache Arrow ecosystem:

- There's a C++ Compute Engine in Arrow that performs computations based on explicit execution plans.
- In Go, this compute engine can be accessed via the C Data Interface (CDATA), allowing Arrow types like arrays and tables to be exposed for computation.
- There's an experimental native Compute Engine in Golang that avoids the need for C-Go or CDATA, offering a more Go-native approach.

## Memory Management and Go's Garbage Collector

I came across an interesting challenge involving Go's garbage collector (GC) and its interaction with C-Go functions. Specifically, when calling a C-Go function, memory is pinned to prevent the GC from freeing or moving it during computation. Once the function completes, the GC might free that memory, which could cause issues if further C/C++ processing is needed. In such cases, it might be worth exploring the use of a C allocator in Golang instead of relying on Go's default allocator.

## Next Steps

To get a more hands-on understanding of Arrow, I'm planning to:

1. Write some basic code using Apache Arrow to familiarize myself with its core functionality.
2. Examine the Golang implementation for opportunities to optimize or introduce Go-specific features, such as using goroutines for concurrency.
3. Explore the GitHub issues page of the Arrow Go repository to identify potential areas where I could contribute.
4. Engage with the Arrow community to better understand the roadmap and discuss future features in the Golang implementation.

## Disclaimer

I'm still in the early stages of this journey, and I realize that my initial understandings might be flawed or not entirely accurate. My knowledge of these components is still developing, and I might not have all the details correct. As a newcomer, I have a lot to learn, and my initial perceptions might change as I gain more experience.

## Conclusion

Even though this will be my first serious attempt at contributing to an open-source project, I'm hopeful that this exploration will provide a solid foundation for understanding the differences between the Java and Golang implementations, as well as some of the internal workings of Arrow's Compute Engine. The next step is to move from theory to practice by writing code, digging into existing issues, and finding the right place to contribute.

This blog post is just the beginning of what I hope will be a series documenting my journey into open-source contribution with this exciting project.
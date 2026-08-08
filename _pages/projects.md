---
permalink: /projects/
title: "Projects"
author_profile: true
layout: single
---

Personal projects I've built or am actively working on.

---

## [Go Kafka](https://github.com/lehoangtran289) *(In Progress)*

A distributed, fault-tolerant message queue built from scratch in Go, inspired by Apache Kafka's design.

- Implements log-based storage, topic partitioning, and consumer group semantics
- Focuses on correctness and understanding distributed consensus under the hood

**Stack:** Go

---

## [Java Single Flight](https://github.com/lehoangtran289/singleflight)

A Java library implementing duplicate suppression (single-flight), similar to Go's `singleflight` package.

- Prevents redundant concurrent calls to the same key from executing multiple times
- Useful for de-duplicating expensive database or network calls under high concurrency

**Stack:** Java

---

## [Java Consistent Hashing](https://github.com/lehoangtran289/consistent-hashing)

A Java library for consistent hashing with virtual node support.

- Minimizes key remapping when nodes are added or removed from the ring
- Supports configurable virtual node replication factors for better load distribution

**Stack:** Java

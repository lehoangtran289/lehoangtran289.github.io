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

## [1BRC - One Billion Row Challenge](https://github.com/lehoangtran289/1brc-java)

My solution to the [One Billion Row Challenge](https://github.com/gunnarmorling/1brc): aggregate min/mean/max temperature per weather station across a 1-billion-row file as fast as possible.

- Optimized for raw throughput on modern Java (JDK 21)
- Includes correctness tests against reference samples and `hyperfine`-based benchmarking

**Stack:** Java

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

---

## [Me2diag - Medical Diagnosis Decision Support System](https://github.com/lehoangtran289/me2diag)

A decision support system for medical diagnosis using Hedge Algebra integrated with Picture Fuzzy Relations, built as the implementation of my published research.

- Published in the *International Journal of Fuzzy Systems* (2023): [doi.org/10.1007/s40815-023-01548-4](https://doi.org/10.1007/s40815-023-01548-4)
- Full-stack system with a Spring Boot backend, React frontend, and a Python Flask model service

**Stack:** Java (Spring Boot), React, Python (Flask), MariaDB, MinIO, Docker

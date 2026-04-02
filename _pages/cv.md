---
layout: archive
# classes: wide
title: "Curriculum Vitae"
permalink: /cv/
author_profile: true
redirect_from:
  - /resume
---


{% include base_path %}

{% include toc %}

---

[View my resume (PDF)]({{ base_path }}/files/Resume_LeHoangTran.pdf)

---

## General Information

- **Full Name:** Le Hoang Tran
- **Languages**: Vietnamese (native), English (professional proficiency)

---

## Education

- M.S. in Computer Science, [George Mason University](https://cec.gmu.edu/) \| Fairfax, VA, USA \| Dec 2026 (expected)
  - Current GPA: 4.0/4.0
- B.S. in Information Technology, [Hanoi University of Science and Technology](https://soict.hust.edu.vn/en/) \| Hanoi, Vietnam \| 2022
  - Excellence Degree (Top 5%) (5 years program)

---

## Work experience

### **[LINE Technology Vietnam](https://vietnamdevcenter.linecorp.com/en) - Backend Software Engineer**  

*Hanoi, Vietnam \| July 2025 – August 2025*

- Researched a **fault-tolerant online data migration** solution for 10TB+ data from MySQL + Cassandra to a unified MySQL architecture using dual-write and backfill strategy.
- Leveraged **AI-assisted development tools** for source code onboarding, automated unit/integration tests, and early-stage code review, increasing development velocity.

### **[Viettel Digital](https://viettel.com.vn/en/) - Backend Software Engineer**

*Hanoi, Vietnam \| May 2021 – June 2025*  

- **Customer Engagement Platform** \| *2023 - 2025*
  - Designed and implemented an **in-house marketing system** to support personalized campaigns via push notifications, in-app messaging, and rewards distribution, delivering **100M+ events** to end users daily.
  - Optimized user segmentation creation **throughput from 500 to 35,000 QPS**, **reducing p95 latency by 80%** by introducing a **Roaring Bitmap-based solution** and multiple caching layers.
  - Engineered a **workflow automation module** using Camunda, including business process orchestration, rule evaluation, and timer events, enabling non-technical operations teams to configure marketing workflows independently.
  - **Led a team of 4 backend engineers**, driving sprint planning, system design, and code reviews to continuously extend system functionality in response to user needs and growing demand.
  - **Monitored system metrics via Grafana** and proactively identified performance improvement opportunities. Reported findings to relevant teams and applied **optimization techniques** including configuration tuning, caching strategies, and microservices patterns.

- **Data Tracking System** \| *2022 - 2023*
  - Designed and delivered a **high-throughput event tracking system** for mobile and web platforms using Java and Spring Boot, processing **150M+ events/day** and **eliminating third-party licensing costs**.
  - Implemented an online tracking rule configuration module, supporting **3,000+ dynamic tracking rules**.
  - **Reduced data-capture latency by 30%** by introducing a decision tree-based engine to optimize the validation process.
  - Engineered reactive data-fetching APIs with **Spring WebFlux** to support high-concurrency workloads, handling over **150M+ events daily**.

- **Smart Authentication System** \| *2021 - 2022*
  - Collaborated on the research and delivery of a **SmartOTP** solution for **20M+ end users**, eliminating reliance on SMS OTPs, **saving over $200K annually**, and enhancing authentication reliability.
  - Implemented secure key exchange, OTP generation, and digital signature mechanisms across critical APIs, strengthening authentication and API security.
  - Applied **mobile security hardening techniques** (code obfuscation, root/jailbreak/tamper detection) to the Android application using native **C/JNI**, passing enterprise-level security audits.

- **User Contact System** \| *2024*
  - Designed a backend module to collect and synchronize user contact data across devices for the Viettel Money application, supporting **20M+ end users**.
  - Optimized contact storage with Roaring Bitmap, **reducing storage usage by 70%** and lowering user matching latency.

---

## Projects

- **Go-Kafka clone**: a distributed fault-tolerant message queue in Go, build from scratch. *In-progress*
- **[Java Single Flight](https://github.com/lehoangtran289/singleflight)**: A easy-to-use Java for duplicate suppression mechanism, similar to the one in Go. *In-progress*
- **[Java Consistent Hashing](https://github.com/lehoangtran289/consistent-hashing)**: A Java library for consistent hashing algorithm, supporting virtual nodes.

---

## Technical Skills

- **Languages:** Java, Go, C, JavaScript, Python
- **Backend:** Spring Framework, Spring Boot, Apache Kafka, Camunda BPM, REST APIs, gRPC, Unit & Integration Testing, Microservices, Distributed Systems
- **Databases:** MariaDB/MySQL, MongoDB, Redis/Valkey, Elasticsearch, MinIO, ClickHouse
- **Infrastructure:** AWS, Docker, Kubernetes, CI/CD, Jenkins, ArgoCD, Helm, SonarQube, Keycloak, ELK, k6
- **Familiar with:** Agile SDLC, Distributed Systems, Design patterns, Cybersecurity, AI-assisted development

---

## Certificates

- **AWS Certified Solutions Architect – Associate** - *Issued Sep 2023*  
  [Credential](https://www.credly.com/badges/99c547c9-b22a-4e07-b640-adfb8224f01e)

- **MongoDB Associate Developer** - *Issued Apr 2024*  
  [Credential](https://www.credly.com/badges/ac338511-9253-48f1-ad78-1cb8f2978d74/public_url)

- **Oracle Certified Associate – Java SE 8 Programmer** - *Issued Apr 2023*  
  [Credential](https://catalog-education.oracle.com/ords/certview/sharebadge?id=EEBB84A0AF12225582E89B5143B601093ABF8EFAF862CC92BE53B8A9F10CC6D5)

- **Redis Certified Developer** - *Issued Dec 2021*  
  [Credential](https://www.credential.net/8c6a1f21-bbcc-45dc-8d0f-805187541b99)

- **English Test**: TOEIC 955/990, IELTS 7.5 (Issued 2024)

---

## Publications

- [Proposed Intelligent Decision Support System using Hedge Algebra integrated with Picture Fuzzy Relations for improvement of decision making in medical diagnoses](https://doi.org/10.1007/s40815-023-01548-4)
  - Hoang, T.L., Pham, H.V., Hung, N.Q. et al. Proposed Intelligent Decision Support System Using Hedge Algebra Integrated with Picture Fuzzy Relations for Improvement of Decision-Making in Medical Diagnoses. Int. J. Fuzzy Syst. 25, 3260–3270 (2023). <https://doi.org/10.1007/s40815-023-01548-4>

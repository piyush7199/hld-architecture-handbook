# 💡 HLD Architecture Handbook: The Intuitive System Designer

## Project Goal

This repository, the **HLD Architecture Handbook**, is designed to be a comprehensive, self-paced learning guide for
mastering **High-Level Design (HLD)** and large-scale system architecture. We focus on providing **intuitive definitions
** and
**in-depth explanations** of core concepts, followed by structured design challenges. The ultimate goal is to help you
understand **the 'Why'** behind every architectural choice—the trade-offs, constraints, and future-proofing
considerations
necessary for building systems at scale.

**Audience:** Engineers with basic programming knowledge looking to transition from small-scale development to designing
highly scalable, reliable, and performant distributed systems.

## 📁 Repository Structure

The content is organized into three progressive categories:

| Folder                           | Category Name        | Focus                                                                                                                                       |
|----------------------------------|----------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| [01-principles](./01-principles) | Core Principles      | Core theoretical concepts: Scale, Availability, CAP Theorem, and foundational architecture styles.                                          |
| [02-components](./02-components) | Components Deep Dive | In-depth analysis of specialized databases, caching, sharding, messaging, and concurrency control.                                          |
| [03-challenges](./03-challenges) | Design Challenges    | Real-world design problems (e.g., URL Shortener, Twitter, E-commerce Flash Sale) applying the concepts learned in the first two categories. |
| README.md                        | (This File)          | The main project index and roadmap.                                                                                                         |

## 🗺️ Learning Roadmap: Core Concepts

We will cover the following topics in sequence before moving to the Design Challenges.

### Category 1: Core Principles (Folder: [01-principles](./01-principles))

| Topic ID | Concept                                                                             |
|----------|-------------------------------------------------------------------------------------|
| 1.1.1    | [CAP Theorem](01-principles/1.1.1-cap-theorem.md)                                   |
| 1.1.2    | [Latency, Throughput, and Scaling](01-principles/1.1.2-latency-throughput-scale.md) |
| 1.1.3    | [Availability and Reliability](01-principles/1.1.3-availability-reliability.md)     |
| 1.1.4    | Data Consistency Models                                                             |
| 1.2.1    | System Architecture Styles                                                          |
| 1.2.2    | Networking Components                                                               |
| 1.2.3    | API Gateway and Service Mesh                                                        |

## Category 2: Components Deep Dive (Folder: [02-components](./02-components))

| Topic ID | Concept                                |
|----------|----------------------------------------|
| 2.1.1    | RDBMS Deep Dive: SQL & ACID            |
| 2.1.2    | NoSQL Deep Dive: The BASE Principle    |
| 2.1.3    | Specialized Databases                  |
| 2.1.4    | Database Scaling                       |
| 2.1.5    | Indexing and Query Optimization        |
| 2.2.1    | Caching Deep Dive                      |
| 2.2.2    | Consistent Hashing                     |
| 2.3.1    | Asynchronous Communication             |
| 2.3.2    | Distributed Transactions & Idempotency |
| 2.4.1    | Rate Limiting Algorithms               |
| 2.4.2    | Consensus                              |

## 🎉 Contributions

We highly encourage community contributions to expand this resource! Before submitting a Pull Request, please read and
follow these guidelines:

### General Guidelines

1. **Clarity and Depth:** Content must maintain the project's goal: providing **intuitive**, easy-to-understand
   definitions while retaining technical **depth**.
2. **Naming Convention:** All new topic files must be placed in the correct category folder (e.g., 01-principles/,
   02-components/) and follow the format: `[ID]-[short-name].md` (e.g., `1.2.1-architecture-styles.md`).

### Template for Adding a New Concept Topic (Category 1 or 2)

Use this structure for any new concept file. The file should provide a clear progression from basic intuition to
technical details.

```
# [ID] Topic Title: Subtitle/Focus

## Intuitive Explanation
[Start with a simple, high-level analogy or definition that a beginner can grasp.]

## In-Depth Analysis
[Dive into the technical specifics, internal workings, and algorithms.]

### Key Concepts / Tradeoffs
* **Concept 1:** ...
* **Tradeoff:** [Discuss the pros/cons of a choice, e.g., speed vs. consistency.]

## 💡 Real-World Use Cases
* [List 2-3 specific examples of companies or scenarios where this concept is applied.]

---

## ✏️ Design Challenge
[Create a concise, open-ended question that forces the reader to apply the concepts from the file.]

```

### Template for Adding a New Design Problem (Category 3)

Use this structure for all design problem files (placed in `03-challenges/`). This format mimics the standard system
design interview process, with a strong emphasis on architectural justification.

```
# [ID] Design a [System Name] (e.g., Design a Twitter Timeline)

## 1. Requirements and Scale

### Functional Requirements (What the system MUST do)
* [e.g., Users must be able to post messages (tweets).]

### Non-Functional Requirements (Constraints/Performance)
* **Scale:** [e.g., 500 Million Daily Active Users (DAU)]
* **QPS:** [e.g., Read QPS: 100k, Write QPS: 5k]
* **Availability:** [e.g., High availability is critical (99.99%)]

## 2. Capacity Estimation and Data Model
[Provide basic calculations for Storage and Bandwidth. Detail the initial database schemas.]

## 3. High-Level Architecture
[A diagram or description of the main components: CDN, LB, API Gateway, Services, Databases.]

## 4. Deep Dive: Architectural Choices and Trade-offs
[This section is critical. For every major component (DB, Caching, Messaging), you MUST explain the choice.]

### Example: Database Choice (SQL vs. NoSQL)
| Choice | Rationale (Why this over the alternative?) | Trade-off / Future Scalability Issue |
| :--- | :--- | :--- |
| **PostgreSQL (SQL)** | Chosen for its ACID properties, which are critical for financial transactions and order integrity. | Vertical scaling limits; future horizontal scaling will require complex application-level sharding. |
| **Cassandra (NoSQL)** | Chosen for extreme read/write performance and horizontal scaling for the social feed data. | **Eventual Consistency** means a user might briefly miss a new post. We accept this trade-off for speed. |

## 5. Failure Handling and Future Scaling
[Discuss how the chosen architecture scales further, what happens if a service fails (fault tolerance), and the key bottlenecks of the final design.]
```
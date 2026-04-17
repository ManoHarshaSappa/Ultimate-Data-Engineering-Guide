# Books, Courses, Certifications & Resources

**Author:** Mano Harsha Sappa | [LinkedIn](https://www.linkedin.com/in/manoharshasappa/) | [Portfolio](https://manoharshasappa.github.io/portfolio_ManoHarshaSappa/) | [Email](mailto:sappamanoharsha@gmail.com)

---

## Contents

1. [Must-Read Books: The Core Three](#1-must-read-books-the-core-three)
2. [All Books by Category](#2-all-books-by-category)
3. [Free Learning Platforms & Courses](#3-free-learning-platforms--courses)
4. [Paid Courses Worth the Investment](#4-paid-courses-worth-the-investment)
5. [Cloud Certifications](#5-cloud-certifications)
6. [Whitepapers & Academic Papers](#6-whitepapers--academic-papers)
7. [Engineering Blogs](#7-engineering-blogs)
8. [Recommended Learning Path](#8-recommended-learning-path)

---

## 1. Must-Read Books: The Core Three

Every serious data engineer must read these three books. They are the foundation.

---

### 1. Fundamentals of Data Engineering
**By:** Joe Reis & Matt Housley | **Publisher:** O'Reilly, 2022

The definitive modern guide to data engineering. Covers the entire data engineering lifecycle: generation, storage, ingestion, transformation, serving. Written by two industry veterans who define the field's current direction.

**Why read it:** Most comprehensive overview of the entire DE landscape. Introduces the Data Engineering Lifecycle framework that's become industry-standard mental model.

**Best for:** Anyone who wants to understand the WHY behind data engineering decisions, not just the HOW.

---

### 2. Designing Data-Intensive Applications (DDIA)
**By:** Martin Kleppmann | **Publisher:** O'Reilly, 2017

The single most important book for understanding how distributed data systems work. Covers databases, replication, partitioning, transactions, batch processing, and stream processing at a fundamental level.

**Why read it:** After reading DDIA, you'll understand WHY Kafka uses a log structure, WHY distributed transactions are hard, and WHY CAP theorem matters in your daily work. Immediate engineering judgment improvement.

**Best for:** Engineers who want to deeply understand the systems they use, not just how to operate them.

---

### 3. Designing Machine Learning Systems
**By:** Chip Huyen | **Publisher:** O'Reilly, 2022

As data engineering increasingly intersects with ML, this book bridges the gap. Covers feature engineering, data pipelines for ML, model deployment, monitoring, and production ML infrastructure.

**Why read it:** ML systems have unique data requirements (feature stores, training/serving skew, data drift). Data engineers increasingly own this infrastructure.

**Best for:** Data engineers who work with ML teams or want to move into ML engineering.

---

## 2. All Books by Category

### Core Data Engineering

| Book | Author | Key Topics |
|------|--------|-----------|
| **Fundamentals of Data Engineering** | Joe Reis, Matt Housley | DE lifecycle, architecture, tool selection |
| **Designing Data-Intensive Applications** | Martin Kleppmann | Distributed systems, databases, replication |
| **97 Things Every Data Engineer Should Know** | Various | Practical wisdom from 97 engineers |
| **Data Pipelines Pocket Reference** | James Densmore | ETL/ELT patterns, pipeline building |
| **Data Management at Scale, 2nd Ed.** | Piethein Strengholt | Enterprise data management |
| **Deciphering Data Architectures** | James Serra | Modern data architecture patterns |

### Spark

| Book | Author | Key Topics |
|------|--------|-----------|
| **Spark: The Definitive Guide** | Bill Chambers, Matei Zaharia | Comprehensive Spark reference (Spark 2.x) |
| **Learning Spark, 2nd Edition** | Jules Damji et al. | Spark 3.x, Delta Lake, structured streaming |
| **High Performance Spark** | Holden Karau, Rachel Warren | Spark optimization, tuning, production |
| **Modern Data Engineering with Apache Spark** | Various | Hands-on Spark + streaming |

### Data Modeling & Warehousing

| Book | Author | Key Topics |
|------|--------|-----------|
| **The Data Warehouse Toolkit, 3rd Ed.** | Ralph Kimball, Margy Ross | Definitive Kimball dimensional modeling |
| **Data Engineering with dbt** | Roberto Zagni | Practical dbt guide |
| **Unlocking dbt** | Christophe Oudar | Advanced dbt patterns |
| **Snowflake Data Engineering** | Allen Cook et al. | Snowflake-specific DE patterns |

### Streaming & Real-Time

| Book | Author | Key Topics |
|------|--------|-----------|
| **Streaming Systems** | Tyler Akidau et al. | Unified batch/streaming theory (Google Dataflow model) |
| **Stream Processing with Apache Flink** | Fabian Hueske, Vasiliki Kalavri | Comprehensive Flink guide |

### Lakehouse & Modern Formats

| Book | Author | Key Topics |
|------|--------|-----------|
| **Delta Lake: The Definitive Guide** | Denny Lee et al. | Delta Lake internals and patterns |
| **Apache Iceberg: The Definitive Guide** | Tomer Shiran et al. | Iceberg format and lakehouse design |
| **Architecting an Apache Iceberg Lakehouse** | Various | Manning guide to Iceberg lakehouse |

### Machine Learning & MLOps

| Book | Author | Key Topics |
|------|--------|-----------|
| **Designing Machine Learning Systems** | Chip Huyen | ML production systems |
| **ML System Design Interview** | Ali Aminian, Alex Xu | ML system design for interviews |
| **The Hundred Page ML Book** | Andriy Burkov | Concise ML theory reference |

### Architecture & Governance

| Book | Author | Key Topics |
|------|--------|-----------|
| **Data Mesh** | Zhamak Dehghani | Domain-oriented decentralized data ownership |
| **Data Governance: The Definitive Guide** | Evren Eryurek et al. | Data catalog, lineage, compliance |
| **Building Evolutionary Architectures, 2nd Ed.** | Neal Ford et al. | Architecture fitness functions, changeability |

### Cloud

| Book | Author | Key Topics |
|------|--------|-----------|
| **Data Engineering with AWS** | Gareth Eagar | AWS data services end-to-end |

### Python & Analysis

| Book | Author | Key Topics |
|------|--------|-----------|
| **Python for Data Analysis, 3rd Ed.** | Wes McKinney | Pandas bible (from the creator of Pandas) |
| **Pandas Cookbook, 3rd Ed.** | Matt Harrison | Practical Pandas recipes |

### Big Data Classics

| Book | Author | Key Topics |
|------|--------|-----------|
| **Hadoop: The Definitive Guide** | Tom White | Hadoop ecosystem reference |
| **Trino: The Definitive Guide** | Matt Fuller et al. | Trino/PrestoSQL deep dive |

---

## 3. Free Learning Platforms & Courses

### Data Engineering Zoomcamp (DataTalks.Club)

**Cost:** Free | **Duration:** 9 weeks

The most complete free DE course available. Covers the entire modern stack:
- Week 1: Docker + Terraform + PostgreSQL
- Week 2: Orchestration (Kestra)
- Week 3: Data Warehouse (BigQuery)
- Week 4: Analytics Engineering (dbt)
- Week 5: Batch Processing (Spark)
- Week 6: Streaming (Kafka)
- Week 9: Capstone Project

**Why:** Cohort-based with Slack community, real project at the end, and highly respected by hiring managers.

### dbt Learn

**Cost:** Free | **Format:** Self-paced

Official dbt courses:
- **dbt Fundamentals** — core concepts, sources, models, tests, docs
- **Jinja, Macros, Packages** — advanced templating
- **Advanced Materializations** — incremental, snapshots, ephemeral
- **Analyses, Seeds, Exposures** — complete project setup

### Databricks Academy

**Cost:** Free (some paid) | **Format:** Self-paced

- Apache Spark Programming with Databricks
- Databricks Fundamentals
- Delta Lake Fundamentals
- Databricks Machine Learning

### Google Cloud Skills Boost (Formerly Qwiklabs)

**Cost:** Free tier available | **Format:** Hands-on labs

- BigQuery fundamentals
- Data Engineering on Google Cloud (learning path)
- Serverless Data Processing with Dataflow

### AWS Skill Builder

**Cost:** Free tier available | **Format:** Self-paced + hands-on

- AWS Data Engineer - Associate learning path
- AWS Glue Immersion Day
- Architecting Data Lakes on AWS

### Coursera / edX

Recommended free-to-audit courses:
- **IBM Data Engineering Professional Certificate** (IBM on Coursera)
- **Data Engineering Foundations Specialization** (IBM on Coursera)

---

## 4. Paid Courses Worth the Investment

| Course | Platform | Focus | Why It's Worth It |
|--------|----------|-------|-------------------|
| **DataExpert.io** | dataexpert.io | Complete DE bootcamp | Zach Wilson's 4/6-week boot camps, real projects, Discord community |
| **LearnDataEngineering.com** | learndataengineering.com | End-to-end DE skills | Andreas Kretz (Cookbook author), comprehensive curriculum |
| **Udemy: The Complete Hands-On Introduction to Apache Airflow** | Udemy | Airflow | Marc Lamberti's course, highest-rated Airflow course |
| **AlgoExpert Data Engineering** | algoexpert.io | DE system design | System design practice for interviews |
| **ByteByteGo** | bytebytego.com | System design | Alex Xu's books and video content |

---

## 5. Cloud Certifications

Certifications signal competence to hiring managers and give you a structured curriculum.

### AWS Certifications

| Certification | Level | Focus | Exam Cost |
|--------------|-------|-------|-----------|
| **AWS Certified Cloud Practitioner** | Foundational | AWS basics | $100 |
| **AWS Certified Data Engineer - Associate** | Associate | DE-specific AWS services | $150 |
| **AWS Certified Solutions Architect - Associate** | Associate | Architecture | $150 |
| **AWS Certified Machine Learning - Specialty** | Specialty | ML on AWS | $300 |

**Recommended path:** Cloud Practitioner → Data Engineer Associate → Solutions Architect Associate

### Google Cloud Certifications

| Certification | Level | Focus | Exam Cost |
|--------------|-------|-------|-----------|
| **Google Cloud Associate Cloud Engineer** | Associate | GCP basics | $200 |
| **Google Professional Data Engineer** | Professional | BigQuery, Dataflow, Pub/Sub | $200 |
| **Google Professional ML Engineer** | Professional | ML on GCP | $200 |

### Azure Certifications

| Certification | Level | Focus | Exam Cost |
|--------------|-------|-------|-----------|
| **Azure Fundamentals (AZ-900)** | Foundational | Azure basics | $165 |
| **Azure Data Engineer Associate (DP-203)** | Associate | Synapse, ADLS, Data Factory | $165 |
| **Azure Data Fundamentals (DP-900)** | Foundational | Data concepts | $165 |

### Databricks Certifications

| Certification | Focus | Cost |
|--------------|-------|------|
| **Databricks Certified Associate Developer for Apache Spark** | PySpark coding | $200 |
| **Databricks Certified Data Engineer Associate** | DE workflows on Databricks | $200 |
| **Databricks Certified Data Engineer Professional** | Advanced DE | $400 |

### dbt Certifications

| Certification | Focus | Cost |
|--------------|-------|------|
| **dbt Analytics Engineering Certification** | dbt modeling and deployment | $200 |

---

## 6. Whitepapers & Academic Papers

The papers that shaped modern data engineering. Reading these gives you insight the tool documentation never provides.

| Paper | Authors | Why Read It |
|-------|---------|-------------|
| **MapReduce: Simplified Data Processing on Large Clusters** | Dean & Ghemawat (Google, 2004) | The paper that started big data. Understanding MapReduce makes Spark's design obvious. |
| **The Google File System** | Ghemawat et al. (Google, 2003) | Foundation of distributed file systems. HDFS was built on this design. |
| **Spark: Cluster Computing with Working Sets** | Matei Zaharia et al. (2010) | The original Spark paper. Explains the RDD concept and why it's faster than MapReduce. |
| **Lakehouse: A New Generation of Open Platforms** | Armbrust et al. (2021) | Academic paper defining the Lakehouse concept — Delta Lake, Iceberg, Hudi. |
| **The Data Lakehouse: Data Warehousing and More** | Various (2023) | Updated academic analysis of lakehouse platforms. |
| **Tidy Data** | Hadley Wickham (2014) | Foundational paper on how data SHOULD be structured. Every DE should read this. |
| **A Five-Layered Business Intelligence Architecture** | Kupfer et al. (2011) | The academic foundation of the modern data stack layers. |
| **XTable in Action: Seamless Interoperability in Data Lakes** | Various (2024) | How Delta, Iceberg, and Hudi can interoperate via XTable. |
| **Building a Universal Data Lakehouse** | Onehouse team | Practical guide to multi-format lakehouse design. |
| **Big Data Quality: A Data Quality Profiling Model** | Various | Academic framework for data quality dimensions. |
| **Lakehouse Paper Summary** | dataengineer.io | Collection of all major DE whitepapers in one place. |

---

## 7. Engineering Blogs

Follow these blogs to stay current. These engineers write about what's actually happening in production.

### Company Engineering Blogs

| Company | Blog URL | Best For |
|---------|----------|---------|
| **Netflix** | [netflixtechblog.com](https://netflixtechblog.com/tagged/big-data) | Iceberg, streaming, scale |
| **Uber** | [eng.uber.com](https://eng.uber.com) | AresDB, ML platform, big data |
| **Databricks** | [databricks.com/blog/engineering](https://www.databricks.com/blog/category/engineering/data-engineering) | Delta Lake, Spark, MLflow |
| **Airbnb** | [medium.com/airbnb-engineering](https://medium.com/airbnb-engineering/data/home) | Data platform, Superset, ML |
| **Amazon AWS** | [aws.amazon.com/blogs/big-data](https://aws.amazon.com/blogs/big-data/) | AWS services, patterns |
| **Microsoft** | [techcommunity.microsoft.com](https://techcommunity.microsoft.com/t5/data-architecture-blog/bg-p/DataArchitectureBlog) | Azure, Fabric, Synapse |
| **Microsoft Fabric** | [blog.fabric.microsoft.com](https://blog.fabric.microsoft.com/) | Microsoft Fabric platform |
| **Meta** | [engineering.fb.com](https://engineering.fb.com/category/data-infrastructure/) | Hive, Presto, Spark at scale |
| **Onehouse** | [onehouse.ai/blog](https://www.onehouse.ai/blog) | Apache Hudi, lakehouse |
| **Estuary** | [estuary.dev/blog](https://estuary.dev/blog/) | CDC, real-time ELT |
| **Oracle** | [blogs.oracle.com/datawarehousing](https://blogs.oracle.com/datawarehousing/) | Data warehouse patterns |

### Individual Creator Blogs

| Creator | Blog/Newsletter | Focus |
|---------|----------------|-------|
| **Joe Reis** | [joereis.substack.com](https://joereis.substack.com) | DE strategy, industry trends |
| **Zach Wilson** | [dataengineer.io](https://blog.dataengineer.io) | Hands-on DE, career |
| **Seattle Data Guy (Ben Rogojan)** | [seattledataguy.substack.com](https://seattledataguy.substack.com) | Practical DE, AWS |
| **Start Data Engineering** | [startdataengineering.com](https://www.startdataengineering.com) | DE patterns, tutorials |
| **Data Engineering Weekly** | [dataengineeringweekly.com](https://www.dataengineeringweekly.com) | Weekly curated DE news |
| **ByteByteGo** | [blog.bytebytego.com](https://blog.bytebytego.com) | System design diagrams |
| **Metadata Weekly** | [metadataweekly.substack.com](https://metadataweekly.substack.com) | Data catalog, governance |

---

## 8. Recommended Learning Path

Based on your current experience level:

### Zero to Junior (0-12 months)

```
Month 1-2: Foundations
  - Book: Python for Data Analysis (Pandas basics)
  - Course: DataTalks.Club Zoomcamp (Module 1-3: Docker + Terraform + SQL)
  - Practice: NY Taxi dataset in PostgreSQL

Month 3-4: SQL & Data Modeling
  - Book: Kimball's Data Warehouse Toolkit (Chapters 1-5)
  - Practice: Build a star schema for a retail dataset
  - Course: dbt Learn (Fundamentals + Jinja)

Month 5-6: Pipeline Orchestration
  - Course: Airflow Fundamentals (Marc Lamberti, Udemy)
  - Practice: Build a pipeline with Airflow + dbt + PostgreSQL
  - Course: DataTalks.Club Zoomcamp (Module 4-5: dbt + BigQuery)

Month 7-9: Big Data Processing
  - Course: DataTalks.Club Zoomcamp (Module 6: Spark)
  - Practice: Reprocess NY Taxi with PySpark
  - Book: Learning Spark, 2nd Edition (first 8 chapters)

Month 10-12: Streaming + Cloud
  - Course: DataTalks.Club Zoomcamp (Module 7: Kafka)
  - Certification: AWS Cloud Practitioner
  - Project: End-to-end portfolio project (see Module 16)
```

### Junior to Mid-Level (1-3 years experience)

```
Read: Designing Data-Intensive Applications (DDIA) — one chapter/week
Read: Fundamentals of Data Engineering — full read

Deepen skills:
  - Spark: High Performance Spark book + tune a real job
  - Kafka: Set up Schema Registry + implement exactly-once
  - dbt: Advanced materializations + custom tests + packages
  - Cloud: Get AWS Data Engineer Associate certification

Practice:
  - Contribute to an open-source project (dbt packages, Airflow providers)
  - Write about what you learn (blog, LinkedIn posts)
  - Do 50 LeetCode SQL Medium problems
```

### Mid-Level to Senior (3-7 years experience)

```
Read: Streaming Systems (Tyler Akidau) — streaming theory
Read: Data Mesh (Zhamak Dehghani) — organizational data architecture

Deepen:
  - Lead a migration project (Hadoop to cloud, ETL to dbt, etc.)
  - Design a data platform from scratch for a use case
  - Read the original papers: MapReduce, GFS, Spark, Lakehouse
  - Get Databricks Certified Data Engineer Professional

Contribute:
  - Speak at a local meetup
  - Write technical blog posts
  - Mentor junior engineers
  - Review architectures cross-team
```

---

## Quick Reference: Minimum Reading List

If you only have time for 5 books, read these in order:

```
1. Designing Data-Intensive Applications (DDIA) — Martin Kleppmann
   → Foundation for all distributed systems thinking

2. Fundamentals of Data Engineering — Joe Reis & Matt Housley
   → Modern DE framework and decision-making

3. The Data Warehouse Toolkit — Ralph Kimball
   → Dimensional modeling (still the dominant pattern)

4. Learning Spark, 2nd Edition — Jules Damji et al.
   → Practical Spark for modern data lakes

5. Designing Machine Learning Systems — Chip Huyen
   → ML production systems (the next frontier for DE)
```

---

← [13 Interview Prep](13-interview-prep.md) | [15 Communities →](15-communities.md)

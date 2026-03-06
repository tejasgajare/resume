# Yardi Systems — Software Engineer

**Company:** Yardi Systems
**Title:** Software Engineer
**Location:** Pune, India
**Duration:** August 2019 — April 2021
**Industry:** Real Estate Technology / Property Management Software

---

## Overview

I worked as a Software Engineer at Yardi Systems, a leading provider of property management
and real estate investment software. This was my first professional engineering role after
completing my B.E. in Computer Engineering. I worked on compliance tooling and enterprise
application development using the Microsoft technology stack.

This role gave me foundational experience in enterprise software development, database design,
regulatory compliance engineering, and working with large-scale production systems.

---

## What I Built

### 1. CCPA & GDPR Compliance Data Scrubbing Tool

Designed and built a data scrubbing tool to ensure compliance with the **California Consumer
Privacy Act (CCPA)** and the **EU General Data Protection Regulation (GDPR)**.

**Technical details:**
- Handled complex **hierarchical database relationships** — Yardi's property management
  databases have deeply nested entity relationships (properties → units → tenants → leases →
  transactions → documents)
- Implemented in **T-SQL** with stored procedures for database-level data scrubbing
- Built a feature to **automatically identify sensitive fields**, reducing field selection
  time by 50% — previously, engineers had to manually identify which fields contained PII
- Addressed cascading delete/anonymize challenges where scrubbing one record could affect
  hundreds of related records across tables

**Technologies:** T-SQL, Microsoft SQL Server, ASP.NET (for the UI layer)

**Impact:**
- Ensured Yardi's property management platform was compliant with CCPA and GDPR before
  enforcement deadlines
- 50% reduction in field selection time for compliance operations
- Handled the complexity of real estate data models where a single tenant's data can span
  dozens of interconnected tables

### 2. Harbor Management System

Developed a **Harbor Management System** for lease tracking and operational management:

- Built with **ASP.NET** and **Microsoft SQL Server**
- Dashboard-based interface for managing harbor/marina leases
- Lease lifecycle tracking: creation, renewal, expiration, billing
- <!-- TODO: Confirm if this was a new product or enhancement to existing Yardi platform -->

**Technologies:** ASP.NET, C#, Microsoft SQL Server, HTML/CSS/JavaScript

---

## Technologies Used

| Category        | Technologies                                          |
|----------------|-------------------------------------------------------|
| Languages       | C#, T-SQL, JavaScript                                |
| Backend         | ASP.NET, ASP.NET MVC                                 |
| Database        | Microsoft SQL Server                                  |
| Frontend        | HTML, CSS, JavaScript <!-- TODO: confirm if Angular/jQuery/etc --> |
| Tools           | Visual Studio, SQL Server Management Studio, Git      |
| Practices       | Stored procedures, database schema design, compliance engineering |

---

## Key Contributions

### Technical Depth

1. **Complex data relationships**: Navigated deeply hierarchical database schemas with dozens
   of interconnected tables — a skill that directly transferred to understanding MongoDB
   document relationships at HPE
2. **Compliance engineering**: Understanding regulatory requirements (CCPA, GDPR) and
   translating them into technical implementations
3. **Performance optimization**: T-SQL stored procedures optimized for large datasets with
   complex joins across hierarchical relationships
4. **Automated sensitive field detection**: Built intelligence into the tool to identify PII
   fields automatically, reducing manual effort by 50%

### Impact Metrics

| Metric                            | Value                                   |
|----------------------------------|-----------------------------------------|
| Field selection time reduction   | 50% (automated PII detection)           |
| Compliance standards addressed   | 2 (CCPA + GDPR)                         |
| Database complexity              | Hierarchical with cascading relationships|
| Duration                         | ~20 months                              |

---

## What I Learned

### Technical Skills
- **Enterprise database design**: Complex relational schemas, stored procedures, query optimization
- **Microsoft stack**: ASP.NET, C#, SQL Server — full-stack enterprise development
- **Compliance/regulatory engineering**: Translating legal requirements into code
- **T-SQL expertise**: Advanced stored procedures for data transformation and scrubbing
- **Hierarchical data handling**: Navigating deeply nested entity relationships

### Professional Growth
- Working in a large enterprise software company with established processes
- Understanding the real estate technology domain
- Shipping production software used by property management companies globally
- Balancing compliance requirements with usability and performance

---

## Connection to Later Work

| Yardi Experience                      | Later Application                                      |
|--------------------------------------|-------------------------------------------------------|
| Database schema design (SQL Server)  | MongoDB schema design at HPE                           |
| Compliance engineering (CCPA/GDPR)   | Global trade compliance at HPE (export controls)       |
| Hierarchical data relationships      | BSON document nesting in MongoDB at HPE                |
| Stored procedures / T-SQL            | Database query optimization at HPE                     |
| ASP.NET web development              | Go REST API development at HPE                         |
| Enterprise software environment      | Fortune 500 environment at HPE                         |
| Data scrubbing / PII handling        | Security mindset applied to tenant isolation at HPE    |

---

## Interview Talking Points

**Why this matters:**
- Demonstrates I can work with complex, real-world database schemas (not just textbook examples)
- Shows experience with regulatory compliance — a growing concern in every industry
- 50% time reduction is a concrete, quantifiable impact
- Enterprise experience before grad school shows professional maturity

**STAR-ready summary:**
- **Situation**: Yardi needed CCPA/GDPR compliance for property management platform with
  complex hierarchical database
- **Task**: Design and build a data scrubbing tool that handles cascading relationships and
  identifies PII fields
- **Action**: Built T-SQL tool navigating hierarchical relationships, added automated
  sensitive field detection
- **Result**: Full CCPA/GDPR compliance, 50% reduction in field selection time, handled
  complex cascading data relationships

---

<!-- TODO: Fill in details about:
  - Specific Yardi products worked on (Voyager? Breeze? RentCafe?)
  - Team size and reporting structure
  - Harbor Management System — was this a standalone product or module?
  - Any involvement in code reviews, deployments, or on-call
  - Technologies beyond what's listed (any CI/CD, testing frameworks, etc.)
  - Number of database tables/entities managed by the scrubbing tool
  - Any awards, recognition, or notable achievements
-->

*Source: Resume*

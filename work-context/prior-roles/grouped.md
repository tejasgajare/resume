# Grouped — Full Stack Developer

**Company:** Grouped
**Title:** Full Stack Developer
**Location:** Syracuse, NY
**Duration:** January 2022 — April 2022
**Type:** <!-- TODO: Confirm — startup? class project? contract? -->

---

## Overview

I led development of core microservices and cloud infrastructure for a social media platform
at Grouped. This was a full-stack role where I designed the architecture from the ground up,
including a three-tier, multi-AZ AWS infrastructure running Spring Boot and Flask microservices.

This role gave me foundational experience in cloud architecture, horizontal scaling, and
full-stack development that directly prepared me for my work at HPE.

---

## What I Built

### 1. Cloud Infrastructure Architecture

Designed and deployed a **three-tier, multi-AZ infrastructure** on AWS:

- **Compute layer**: Spring Boot (Java) and Flask (Python) microservices
- **Container orchestration**: AWS ECS with Fargate for serverless container management
- **Scaling**: 5 horizontally scaled Fargate instances behind an Application Load Balancer
- **High availability**: Multi-AZ deployment for fault tolerance

**Architecture:**
```
Internet → ALB → ECS Fargate (5 instances, multi-AZ)
                  ├── Spring Boot services (Java)
                  └── Flask services (Python)
                      ↓
                  RDS / Data layer
```

### 2. Core Microservices

Led development of the backend microservice layer:

- **Spring Boot microservices** (Java): Core business logic, user management, API layer
- **Flask microservices** (Python): Supporting services, data processing
- Service-to-service communication patterns
- RESTful API design for frontend consumption

### 3. Server-Side Rendering Optimization

Implemented server-side rendering that **decreased page load time by 35%**:

- Moved rendering logic from client to server for initial page loads
- Reduced time-to-first-paint and improved SEO
- Balanced SSR with client-side interactivity for optimal user experience

---

## Technologies Used

| Category        | Technologies                                         |
|----------------|------------------------------------------------------|
| Languages       | Java, Python, JavaScript/TypeScript                  |
| Backend         | Spring Boot, Flask                                   |
| Cloud           | AWS ECS, Fargate, ALB, RDS, Multi-AZ                |
| Frontend        | <!-- TODO: React? Vue? Angular? Confirm framework --> |
| Infrastructure  | AWS (ECS, Fargate, RDS, VPC, ALB)                   |
| DevOps          | Docker, CI/CD <!-- TODO: confirm pipeline tools -->  |

---

## Key Contributions

### Architecture Decisions

1. **ECS + Fargate over EC2**: Chose serverless containers to reduce operational overhead
   and enable automatic scaling without managing underlying instances
2. **Multi-AZ deployment**: Ensured high availability from day one, not as an afterthought
3. **Polyglot microservices**: Used Spring Boot for heavyweight services and Flask for
   lightweight supporting services, choosing the right tool for each job
4. **Horizontal scaling**: Designed stateless services that could scale horizontally behind
   a load balancer

### Impact Metrics

| Metric                          | Value                    |
|--------------------------------|--------------------------|
| Page load time improvement     | 35% reduction via SSR    |
| Fargate instances              | 5 horizontally scaled    |
| Availability zones             | Multi-AZ deployment      |
| Microservice frameworks        | 2 (Spring Boot + Flask)  |

---

## What I Learned

### Technical Skills
- **AWS infrastructure**: Hands-on experience with ECS, Fargate, ALB, RDS, VPC configuration
- **Microservices architecture**: Designing service boundaries, inter-service communication
- **Horizontal scaling**: Stateless service design, load balancing strategies
- **Full-stack development**: End-to-end from infrastructure to API to frontend
- **SSR optimization**: Server-side rendering tradeoffs and implementation

### Professional Growth
- Leading development of a platform from scratch (architecture → implementation → deployment)
- Making technology choices and defending them (Fargate vs EC2, polyglot vs monolingual)
- Working in a fast-paced startup-like environment during grad school

---

## Connection to Later Work

| Grouped Experience                    | HPE Application                                        |
|--------------------------------------|-------------------------------------------------------|
| AWS ECS/Fargate                      | AWS S3, IAM, EKS at HPE                               |
| Multi-AZ high availability           | Multi-env (dev/intg/prod) deployments at HPE           |
| Spring Boot microservices            | Go microservices at HPE (same patterns, different lang)|
| Flask services                       | Python FastAPI wellness proxy at HPE                   |
| Load balancer + horizontal scaling   | K8s HPA + Istio load balancing at HPE                  |
| Docker containers                    | Docker → K8s container orchestration at HPE            |

---

## Interview Talking Points

**Why this matters:**
- Demonstrates I can architect systems from the ground up, not just work on existing ones
- Shows leadership — I led development, not just contributed
- Multi-language backend (Java + Python) shows language flexibility
- Quantifiable impact: 35% page load improvement, 5-instance horizontally scaled system

**STAR-ready summary:**
- **Situation**: Social media platform needed scalable backend and infrastructure
- **Task**: Lead development of microservices architecture and cloud deployment
- **Action**: Designed multi-AZ AWS ECS/Fargate infra with Spring Boot + Flask services,
  implemented SSR optimization
- **Result**: 35% page load time reduction, horizontally scaled to 5 Fargate instances,
  multi-AZ high availability

---

<!-- TODO: Fill in details about:
  - The social media platform's purpose/target audience
  - Specific API endpoints or features built
  - Frontend framework used (React? Vue? Angular?)
  - Database choice (RDS — PostgreSQL? MySQL?)
  - Was this a startup, grad school project, or contract role?
  - Team size and my specific leadership responsibilities
  - Any demo or launch metrics
-->

*Source: Resume*

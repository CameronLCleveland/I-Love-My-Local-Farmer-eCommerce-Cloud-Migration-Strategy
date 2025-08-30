# I Love My Local Farmer: eCommerce Cloud Migration Strategy

**Author:** Cameron Cleveland
**Date:** 08/30/2025
**Project:** Joint Migration Class Homework: Improve the Medium Article

## Project Overview
This project analyzes and improves upon the cloud migration strategy described in the Medium article "How we migrated our eCommerce application to the cloud in near zero downtime" by Karim Ichallamene. The original project involved replatforming a monolithic Magento application from on-premises to AWS with a goal of near-zero downtime. This document outlines the original approach, then provides a detailed analysis and modernized, improved strategy.

### Goals (Based on Article)
- Achieve near-zero downtime migration.
- Complete the migration project in under 4 weeks.
- Replatform to managed AWS services (RDS, OpenSearch, ECS).
- Eliminate risk of data loss during database migration.
- Ensure a seamless cutover for end-users.

## Current State Analysis

The following diagram illustrates the application's original on-premises architecture as described in the article.

![On-Premises eCommerce Architecture](on-premises-before.png)

### Diagram Explanation & Key Issues:
*   **Monolithic Architecture:** The Magento application, MariaDB database, and Elasticsearch cluster were tightly coupled, preventing independent scaling and increasing complexity.
*   **Single Points of Failure:** The standalone database and search servers were critical failure points. An outage would result in full application downtime and potential data loss.
*   **Manual Processes:** Scaling, backups, and deployments were manual operations, consuming valuable DevOps resources and slowing down feature development.
*   **Capacity Limits:** The infrastructure was physically limited by on-premises hardware, hindering global expansion and the ability to handle traffic spikes.

## Future State Recommendation: AWS Architecture (Article's Plan)

The article's proposed future state on AWS involved a replatforming strategy, moving to managed services but maintaining a similar application structure.

![AWS Cloud Architecture](aws-article-after.png)

### Diagram Explanation & Choices:

**1. Compute & Application:**
*   **Amazon ECS on Fargate:** The containerized Magento application runs on a serverless compute engine, eliminating the need to manage EC2 instances and simplifying scaling.
*   **Application Load Balancer:** Distributes incoming traffic across multiple ECS tasks, ensuring high availability and fault tolerance.

**2. Data & Search:**
*   **Amazon RDS for MariaDB:** The database was replatformed to a managed RDS instance in a Multi-AZ configuration, providing automated backups, patching, and built-in high availability with a standby replica.
*   **Amazon OpenSearch Service:** The Elasticsearch cluster was migrated to its managed AWS counterpart, reducing operational overhead.

**3. Migration Tooling:**
*   **Custom Scripts with mydumper/myloader:** The team chose these native database tools over AWS DMS for the homogeneous (MariaDB-to-MariaDB) migration, prioritizing familiarity. An EC2 instance was used as a staging host for the data.

## Task 2: Pain Point Analysis & Solutions (Article's Approach)

### **1. Pain Point: Complex, Risky Database Migration**

The most critical challenge was migrating the 500 GB MariaDB database with zero data loss and minimal downtime. The team's solution was a custom process using `mydumper`/`myloader` for multi-threaded logical backups and native binary log replication for ongoing data sync. This required significant performance tuning on RDS and manual management of the replication process.

**Net Positive?** Yes, but fragile. They achieved their goal but introduced unnecessary complexity and risk by avoiding managed services, requiring deep expertise in MariaDB replication.

### **2. Pain Point: DNS Cutover and Client Caching**

A major discovery was that DNS Time-to-Live (TTL) settings are often ignored by browsers and operating systems, threatening the zero-downtime goal. Their mitigation was to implement a HTTP 301 redirect on their old servers to force traffic to a new subdomain pointing to AWS.

**Net Positive?** Yes, it worked. However, it was a reactive solution that added a manual step to the cutover process. A more robust, proactive traffic shaping strategy was needed.

### **3. Pain Point: Manual and Undocumented Process**

The migration relied on a manual checklist and a "war room" full of experts to execute the cutover. The process, while documented, was not automated or codified, making it difficult to reproduce and introducing the potential for human error.

**Net Positive?** No. This was the biggest gap in their approach. The lack of automation is the primary area for improvement, as it directly contradicts modern DevOps practices and increases risk.

## Task 3: Improved Modernization & Automation Strategy

This section outlines a improved, automated approach to the migration, addressing the shortcomings of the original plan.

### **1. Improvement: Automated, Managed Database Migration**

**Problem:** The custom `mydumper`/`myloader` and binary log replication process was complex and manually intensive.

**Improved Solution: Use AWS Database Migration Service (DMS)**
*   **How it improves the config:** DMS is a fully managed service that handles the initial full load and ongoing replication (CDC) automatically. It provides built-in monitoring via CloudWatch and can optionally perform data validation to ensure integrity.
*   **Drawbacks:** Requires learning a new AWS service and its configuration schema.
*   **Net Positive:** **Yes.** It significantly reduces complexity, eliminates the need for a temporary EC2 instance, and provides a more robust and monitorable replication pipeline, directly de-risking the project.

### **2. Improvement: Full Infrastructure & Pipeline Codification**

**Problem:** The process was manual, relying on checklists and war rooms.

**Improved Solution: Implement a CI/CD Pipeline with IaC**
*   **Tools:** Terraform + Jenkins + Ansible
*   **How it improves the config:**
    *   **Terraform:** Provisions the entire AWS environment (VPC, RDS, ECS, ALB, DMS) from code. This makes the setup reproducible, version-controlled, and disposable.
    *   **Jenkins:** Orchestrates the entire migration as a pipeline. Stages include: `Build Environment with Terraform`, `Start DMS Replication`, `Run Pre-Cutover Tests`, `Execute Cutover`.
    *   **Ansible:** Automates the configuration of any necessary software (e.g., the HTTP 301 redirect on on-prem servers during cutover) and applies optimal RDS parameter tuning.
*   **Net Positive:** **Yes.** This transforms the migration from a manual event into a repeatable, automated procedure, drastically reducing the potential for human error and enabling a smoother, faster rollback if needed.

### **3. Improvement: Safer Traffic Cutover with Blue/Green Deployment**

**Problem: The "flag-day" DNS switch combined with client-side caching was a risk.

**Improved Solution: Blue/Green Deployment with Route 53 Weighted Routing**
*   **How it improves the config:** Instead of a hard cutover, use Route 53 to gradually shift traffic from the on-premises (Blue) environment to AWS (Green).
    1.  Start with 100% of traffic weighted to on-prem.
    2.  Shift 1% of traffic to AWS to monitor for errors.
    3.  Gradually increase the weight to 50%, then 100% over a period of minutes or hours.
*   **Drawbacks:** Requires slightly more complex DNS configuration.
*   **Net Positive:** **Yes.** This provides a built-in, instant rollback mechanism (just shift weights back to 100% on-prem) and allows for real-world performance testing with live traffic, making the cutover infinitely safer and more controlled.

## Conclusion

The original article successfully achieved its goal of a near-zero downtime migration by skillfully applying native database tools. However, by embracing a fully automated pipeline built on **Terraform, Jenkins, and AWS DMS**, and implementing a safer **blue/green cutover strategy**, the same migration could be performed with significantly reduced risk, greater reproducibility, and a stronger foundation for future modernization efforts. This improved strategy aligns with modern DevOps principles and is the recommended approach for any mission-critical cloud migration.

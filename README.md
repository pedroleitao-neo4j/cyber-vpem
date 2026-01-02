# A Vulnerability Prioritization and Exposure Management (VPEM) Graph Use Case

## Introduction

This notebook demonstrates how to create a **Vulnerability Prioritization and Exposure Management (VPEM)** graph in Neo4j using a combination of infrastructure and threat intelligence data. The VPEM graph helps organizations move beyond simple severity scores (like CVSS) to prioritize vulnerabilities based on their **real-world reachability** and **potential business impact**.

This is a classic use case for cybersecurity teams looking to evolve from reactive "patching everything" to a proactive, risk-based vulnerability management strategy.

This example includes two main notebooks:

1. **[Data Ingestion](loader.ipynb):** Loading synthetic data representing organizational assets, vulnerabilities from NVD, and threat intelligence from CISA KEV into Neo4j.
2. **[Specific Use Cases](vpem.ipynb):** Demonstrating key VPEM queries, including:
   - **Reachability Analysis:** Identifying which vulnerabilities are exposed to the internet.
   - **Impact Analysis:** Determining what critical assets are at risk if a vulnerability is exploited.
   - **Contextual Risk Scoring:** Calculating a dynamic risk score that combines technical severity with business context.

### Data Sources

In our example, we integrate three distinct layers of data to build a holistic risk view:

1. **Organizational Asset Context:** We will load synthetic data representing **Applications, Libraries, Build Artifacts, and Cloud Infrastructure** (such as Compute Instances and S3 Buckets). This layer includes the critical "Blast Radius" relationships, such as IAM Policies and Network Interfaces.
2. **Vulnerability Intelligence (NVD):** We will ingest official CVE data from the [NVD database](https://nvd.nist.gov/), including CVSS scores, attack vectors, and CWE problem types, sourced from both CNA and ADP containers.
3. **Threat Intelligence (CISA KEV):** We will enrich our CVE nodes with the [CISA Known Exploited Vulnerabilities (KEV)](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) catalog to flag vulnerabilities that are currently being leveraged by threat actors in the wild.

### The Graph Advantage

Traditional vulnerability management relies on flat lists and spreadsheets, which often lead to "alert fatigue" because they treat a critical vulnerability on an isolated internal server the same as one on an internet-facing gateway.

By using Neo4j, we can perform **Attack Path Analysis** to answer critical questions:

* **Reachability:** Is this vulnerable library actually running on a server accessible from the public internet?
* **Impact:** If this server is compromised, what "Crown Jewel" assets (like PII databases) does its Identity have permission to access?
* **Efficiency:** Which single library update will remediate the highest aggregate risk across our entire production environment?

### The Resulting Schema

With the data loaded, our graph schema will look like the following.

![VPEM Graph Schema](vpem-schema.png)

We include nodes for Applications, Libraries, Build Artifacts, Compute Instances, IAM Policies, and CVEs. The relationships capture deployment, usage, and vulnerability identification.

## Architecture

### Overview

In our VPEM solution, the system is built on a **Security Knowledge Graph** architecture. Unlike traditional relational databases that struggle with deep path traversal (e.g., finding a path from a CVE to an S3 bucket), Neo4j allows us to map the complex relationships between code, infrastructure, and identity in real-time.

### Data Sources & Integration

To build a comprehensive exposure map, the system would typically ingest data from three primary domains:

| Domain | Data Sources | Key Entities Extracted |
| --- | --- | --- |
| **External Threat Intel** | NVD (CVE), CISA KEV, EPSS | Vulnerabilities, Exploit Status, CWEs |
| **Software Supply Chain** | SBOMs, GitHub/GitLab, JFrog Artifactory | Repos, Libraries, Build Artifacts |
| **Cloud Infrastructure** | AWS/Azure/GCP APIs, Wiz, Prisma Cloud | Compute, S3, IAM Roles, Subnets, Endpoints |

### Typical Data Flow

The architecture would follow a standard **Extract, Load, Transform (ELT)** pattern tailored for graph structures:

1. **Ingestion (Python):** Specialized collectors fetch raw JSON data from NVD and CISA. Simultaneously, internal scanners export infrastructure and application manifests.
2. **Entity Resolution (Neo4j MERGE):** Data is streamed into Neo4j using `MERGE` logic. This ensures that if a vulnerability (e.g., Log4j) is found in multiple libraries, it is represented as a single node with multiple relationships, preventing data silos.
3. **Contextual Enrichment:** Once the base nodes are loaded, the system runs graph algorithms to calculate "reachability." It looks for paths between `Endpoint` nodes (the internet) and `ComputeInstance` nodes containing vulnerabilities.
4. **Prioritization Engine:** A final scoring pass is performed. The engine combines the technical severity (CVSS) with the business context (Asset Criticality) and the blast radius (IAM Permissions) to generate a **Contextual Risk Score**.

Depending on an organization's needs, this architecture can be extended with additional data sources, integration points, etc.

### System Integration

The output of this architecture is typically consumed by:

* **Security Operations (SIEM/SOAR):** To prioritize incoming alerts based on the "Blast Radius" of the affected asset.
* **Engineering Teams:** To receive curated "Top 10" lists of library updates that provide the maximum risk reduction for their specific repositories.
* **Compliance & Audit:** To visualize and report on the "Aging" of vulnerabilities on critical PII-bearing systems.


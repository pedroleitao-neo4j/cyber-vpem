# A Cybersecurity Vulnerability Prioritization and Exposure Management (VPEM) Graph Use Case

![A Vulnerability Prioritization and Exposure Management (VPEM) Graph Use Case](headline.png)

This repository demonstrates how to create a **Vulnerability Prioritization and Exposure Management (VPEM)** graph in Neo4j for Cybersecurity applications. Using a combination of infrastructure and threat intelligence data, the VPEM graph helps organizations move beyond simple severity scores (like [CVSS](https://www.first.org/cvss/)) to prioritize vulnerabilities based on their **real-world reachability** and **potential business impact**.

This is a classic use case for cybersecurity teams looking to evolve from reactive "patching everything" to a proactive, risk-based vulnerability management strategy.

![Vulnerability to Compute Instance Reachability](vulnerability-to-compute-graph.png)

This example includes three main notebooks:

1. **[Data Ingestion](loader.ipynb):** Loading synthetic data representing organizational assets, vulnerabilities from [NVD](https://nvd.nist.gov/), and threat intelligence from [CISA KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) into Neo4j.
2. **[Specific Use Cases](vpem.ipynb):** Demonstrating key VPEM queries, including:
   - **Reachability Analysis:** Identifying which vulnerabilities are exposed to the internet.
   - **Impact Analysis:** Determining what critical assets are at risk if a vulnerability is exploited.
   - **Contextual Risk Scoring:** Calculating a dynamic risk score that combines technical severity with business context.
3. **[Other](other.ipynb):** Additional queries and analyses that can be performed on the VPEM graph.

### What is VPEM and Why Does It Matter?

Modern organizations face thousands of potential software security flaws (vulnerabilities) across their systems. It is practically impossible for security teams to fix every single issue immediately.

**Vulnerability Prioritization and Exposure Management (VPEM)** is a strategic approach designed to solve the problem of "alert fatigue." Instead of treating every vulnerability with a high generic severity score as an emergency, VPEM uses real-world context to determine what actually poses a risk to your specific business.

It shifts the focus from "What is broken?" to "What could actually hurt us right now?" by considering factors like:

* **Exposure:** Is the vulnerable system accessible from the public internet, or buried deep inside a private network?
* **Threat Intelligence:** Are hackers actively using this specific flaw in the real world currently?
* **Asset Criticality:** Is the flaw on a test server that holds no data, or on the main production database?

By connecting these dots, VPEM helps organizations stop wasting time patching theoretical problems and focus their limited resources on the handful of critical issues that could lead to a real breach.

### Data Sources

In our example, we integrate three distinct layers of data to build a holistic risk view:

1. **Organizational Asset Context:** We will load synthetic data representing **Applications, Libraries, Build Artifacts, and Cloud Infrastructure** (such as Compute Instances and S3 Buckets). This layer includes the critical "Blast Radius" relationships, such as IAM Policies and Network Interfaces.
2. **Vulnerability Intelligence (NVD):** We will ingest official CVE data from the [NVD database](https://nvd.nist.gov/), including CVSS scores, attack vectors, and CWE problem types, sourced from both CNA and ADP containers.
3. **Threat Intelligence (CISA KEV):** We will enrich our CVE nodes with the [CISA Known Exploited Vulnerabilities (KEV)](https://www.cisa.gov/known-exploited-vulnerabilities-catalog) catalog to flag vulnerabilities that are currently being leveraged by threat actors in the wild.

### The Graph Advantage

Traditional vulnerability management relies on flat lists and spreadsheets, which often lead to "alert fatigue" because they treat a critical vulnerability on an isolated internal server the same as one on an internet-facing gateway.

By using Neo4j, we can perform **[Attack Path Analysis](https://github.com/pedroleitao-neo4j/cyber-apa)** to answer critical questions:

* **Reachability:** Is this vulnerable library actually running on a server accessible from the public internet?
* **Impact:** If this server is compromised, what "Crown Jewel" assets (like PII databases) does its Identity have permission to access?
* **Efficiency:** Which single library update will remediate the highest aggregate risk across our entire production environment?

### The Resulting Schema

With the data loaded, our graph schema will look like the following.

![VPEM Graph Schema](vpem-schema.png)

We include nodes for Applications, Libraries, Build Artifacts, Compute Instances, IAM Policies, and CVEs. The relationships capture deployment, usage, and vulnerability identification.

#### The Schema

This Neo4j schema is conceptually aligned with common cybersecurity ontologies (e.g., the [UOC Cyber ontology](https://unifiedcyberontology.org/)) in terms of core entities and relationships. It models Vulnerabilities and Weaknesses, Software/Products and Components, Hosts/Endpoints, Identities/Policies, Data Services, and Threat Intelligence catalogs. The relationship semantics (e.g., affects, identified-in, authenticates-via, resolves-to, assumes, has-access-to) reflect typical attack-path and blast-radius reasoning patterns. While not a 1:1 OWL import, labels and relationships can be straightforwardly mapped to UOC classes/properties; directionality and cardinalities are chosen to support efficient path traversal and prioritization.

Node labels and properties (as created in `loader.ipynb`):
- CVE: id (unique), state, assigner, publishedDate, lastModifiedDate, source, description, baseScore, baseSeverity, attackComplexity, attackVector, availabilityImpact, confidentialityImpact, integrityImpact, privilegesRequired, userInteraction, kev_addedDate, kev_dueDate, kev_reason, isKnownExploited
- CWE: id (unique)
- Vendor: name (unique)
- Product: name (unique)
- Library: name, version, language
- BuildArtifact: id, registry
- Repo: name, criticality
- Application: name, tier
- ComputeInstance: id, name, public_ip
- Endpoint: url
- Identity: name, arn
- IAMPolicy: name
- CloudService: name, resource_name
- Team: name
- Catalog: name, lastUpdated, version

Schema outline in Cypher:

```cypher
// Uniqueness constraints
CREATE CONSTRAINT cve_id_unique IF NOT EXISTS FOR (c:CVE) REQUIRE c.id IS UNIQUE;
CREATE CONSTRAINT cwe_id_unique IF NOT EXISTS FOR (w:CWE) REQUIRE w.id IS UNIQUE;
CREATE CONSTRAINT vendor_name_unique IF NOT EXISTS FOR (v:Vendor) REQUIRE v.name IS UNIQUE;
CREATE CONSTRAINT product_name_unique IF NOT EXISTS FOR (p:Product) REQUIRE p.name IS UNIQUE;

// Vulnerabilities, weaknesses, vendors, products
MERGE (v:CVE {id:$cveId})
SET v += {
   state:$state,
   assigner:$assigner,
   publishedDate:$publishedDate,
   lastModifiedDate:$lastModifiedDate,
   source:$source,
   description:$description,
   baseScore:$baseScore,
   baseSeverity:$baseSeverity,
   attackComplexity:$attackComplexity,
   attackVector:$attackVector,
   availabilityImpact:$availabilityImpact,
   confidentialityImpact:$confidentialityImpact,
   integrityImpact:$integrityImpact,
   privilegesRequired:$privilegesRequired,
   userInteraction:$userInteraction,
   kev_addedDate:$kev_addedDate,
   kev_dueDate:$kev_dueDate,
   kev_reason:$kev_reason,
   isKnownExploited:$isKnownExploited
}
MERGE (w:CWE {id:$cweId})
MERGE (v)-[:HAS_PROBLEM_TYPE]->(w)

MERGE (vend:Vendor {name:$vendor})
MERGE (prod:Product {name:$product})
MERGE (vend)-[:PROVIDES]->(prod)
MERGE (v)-[:AFFECTS]->(prod)

// Supply chain and deployment
MERGE (lib:Library {name:$library, version:$version})
SET lib.language = $language
MERGE (ba:BuildArtifact {id:$artifactId})
SET ba.registry = $registry
MERGE (repo:Repo {name:$repo})
SET repo.criticality = $criticality
MERGE (app:Application {name:$app})
SET app.tier = $tier
MERGE (host:ComputeInstance {id:$instanceId})
SET host += {name:$instanceName, public_ip:$publicIp}
MERGE (end:Endpoint {url:$endpoint})

MERGE (lib)-[:DEPENDENCY_OF]->(ba)
MERGE (ba)-[:BUILT_FROM]->(repo)
MERGE (ba)-[:RUNNING_AS]->(app)
MERGE (app)-[:HOSTED_ON]->(host)
MERGE (end)-[:RESOLVES_TO]->(host)
MERGE (v)-[:IDENTIFIED_IN]->(lib)

// Identity, access, and data services
MERGE (id:Identity {name:$identity})
SET id.arn = $identityArn
MERGE (pol:IAMPolicy {name:$policy})
MERGE (svc:CloudService {name:$service, resource_name:$resource})

MERGE (app)-[:AUTHENTICATES_VIA]->(id)
MERGE (host)-[:RUNS_AS]->(id)
MERGE (id)-[:ASSUMES]->(pol)
MERGE (pol)-[:HAS_ACCESS_TO]->(svc)

// Threat intelligence catalog and ownership
MERGE (cat:Catalog {name:'CISA KEV'})
SET cat += {lastUpdated:$kevLastUpdated, version:$kevVersion}
MERGE (team:Team {name:$team})
MERGE (v)-[:LISTED_IN]->(cat)
MERGE (team)-[:MANAGES]->(id)
```

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

### Outward System Integration

The VPEM graph acts as a **central nervous system** for security data. Its value is fully realized when the prioritized insights are pushed back into the tools where developers and security analysts spend their time.

#### Security Operations (SIEM/SOAR)

Modern SIEMs (like Splunk or Sentinel) are often overwhelmed by alerts. Integrating the VPEM graph allows for **Contextual Alert Enrichment**:

* **Dynamic Severity:** When an IDS/IPS detects an attack, the SOAR platform queries Neo4j to see if the targeted asset is actually vulnerable to that specific exploit and if it has a high "Blast Radius."
* **Automated Containment:** If a "Scenario C" (Recent KEV + Public IP) vulnerability is detected, the SOAR can automatically trigger an isolation policy in the Cloud environment until a patch is applied.

#### Engineering & DevOps (Ticketing & CI/CD)

To bridge the gap between Security and Engineering, the graph provides **Remediation Intelligence**:

* **Jira/GitHub Issues:** Instead of bulk-exporting 1,000 vulnerabilities, the system creates a single ticket for a Repository owner. The ticket identifies the specific **Library** that needs an upgrade to resolve the highest number of reachable vulnerabilities.
* **Policy as Code (Guardrails):** The CI/CD pipeline can query the graph during a build. If a new deployment would create a "Lateral Movement" path to a Crown Jewel (P0) asset, the build can be automatically failed.

#### Vulnerability Management (VM) & GRC

For compliance and executive reporting, the graph provides **Strategic Risk Metrics**:

* **SLA Tracking:** Integration with GRC tools (like Archer or ServiceNow) allows for tracking "Vulnerability Aging" specifically on critical assets, ensuring the organization meets CISA KEV remediation deadlines.
* **Risk Dashboarding:** Business leaders receive high-level views of "Aggregate Risk by Business Unit," allowing them to allocate security budget to the teams or products with the highest contextual exposure.

#### Real-time Notifications (ChatOps)

Immediate alerts for "High-Urgency" scenarios ensure that critical threats are handled within minutes:

* **Slack/Teams Integration:** Pushing "Scenario C" alerts directly to the #AppSec or #Incident-Response channels when a newly exploited CVE is detected on an internet-facing production instance.


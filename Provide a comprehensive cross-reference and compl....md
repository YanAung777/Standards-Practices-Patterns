# **NZISM Compliance Mapping: Microsoft Foundry AI Security Framework**

**Prepared by:** Senior Security Architect  
**To:** Enterprise Architecture & Governance Board / C\&A Authorities  
**Status:** Formal Accreditation Reference Material  
**Classification:** In-Confidence  
**Primary Alignment:** New Zealand Information Security Manual (NZISM) v3.x Baseline Controls

## **Executive Context**

When accrediting an Artificial Intelligence system under the **NZISM Chapter 4 (System Certification and Accreditation)** framework, the risk owner (Agency Chief Executive) must have assurance that the introduction of non-deterministic Large Language Models (LLMs) does not degrade the agency's overall security posture.  
Traditional IT controls evaluate static code and fixed data pathways. Generative AI systems, however, introduce dynamic execution risks such as prompt injections, algorithmic bias, data leakage, and structural application crashes.  
To bridge this gap, this document maps **Microsoft Foundry’s** native capabilities and the **Agent Identity** architecture directly to mandated NZISM controls. It provides the necessary evidence base for your agency's **Service Security Certificate (SSC)**.

## **NZISM to Microsoft Foundry Mapping Matrix**

The following matrix provides a direct cross-reference between NZISM baseline control requirements and their corresponding technical implementations within a hardened Microsoft Foundry enterprise deployment.

| NZISM Chapter & Control | Mandated Policy Objective | Microsoft Foundry Component / Control Architecture | Compensating / Implementation Strategy |
| :---- | :---- | :---- | :---- |
| **Chapter 16: Security Logging & Auditing** • *Control 16.6.1 (Event Logging)* • *Control 16.6.3 (Audit Trail Retention)* | Maintain an immutable, non-repudiable log of system events, security violations, and privileged access to answer: *What happened, when, and who did it?* | **Foundry Diagnostic Streams \+ Microsoft Sentinel Integration** | All 4 Intervention Points (Input, Tool Call, Tool Response, Output) emit telemetry directly into Azure AppTraces. KQL Parsers normalize block/mask events, prompt risk scores, and content classification tokens for ingestion into the central SOC. |
| **Chapter 14: Access Control & IAM** • *Control 14.1.1 (Least Privilege)* • *Control 14.3.1 (Identification & Auth)* | Restrict access to systems and information assets to authorized users and processes, strictly bounded by operational necessity. | **Entra ID User-Assigned Managed Identities (Agent IDs)** | Autonomous agents are provisioned with dedicated Managed Identities (Service Principals) bounded by custom Azure RBAC roles. They possess an isolated execution identity completely divorced from the end-user's broader tenant privileges. |
| **Chapter 17: Software Security & Inputs** • *Control 17.1.5 (Input Validation)* • *Control 17.2.10 (Data Sanatisation)* | Validate and sanitize all external inputs to prevent injection attacks, system hijacking, buffer overflows, or unexpected application states. | **Foundry Content Safety & Runtime Guardrails (Prompt Shield)** | **Intervention Point 1 (User Input)** acts as an inline input validation layer. It runs real-time heuristic and embedding-based checks to detect, intercept, and block direct system prompt overrides (Jailbreaks) *before* tokens are passed to the model inference API. |
| **Chapter 23: Cloud Computing** • *Control 23.2.1 (Data Boundary Isolation)* • *Control 23.5.3 (Cloud Alerts)* | Ensure government data remains isolated within approved jurisdictional boundaries and network enclaves. Generate alerts on cloud anomalies. | **Azure Private Link, VNets, and Automated Azure Policy Initiatives** | Foundry Hubs and Projects are enclosed within Azure Virtual Networks utilizing Private Endpoints. Public internet ingress and egress are completely blocked. System evaluation metrics and Content Safety blocks trigger real-time high-severity alerts in Defender for Cloud and Sentinel. |
| **Chapter 15: Cryptography** • *Control 15.2.1 (Data at Rest Encryption)* • *Control 15.3.1 (Data in Transit)* | Protect information confidentiality and integrity through approved cryptographic mechanisms during storage and transit. | **Azure Storage Service Encryption & TLS 1.3 Bounding** | Fine-tuning weights, custom vector store embeddings (Azure AI Search), and index data are encrypted using **Customer-Managed Keys (CMK)** via Azure Key Vault. All API endpoints and intra-service tool calls strictly enforce TLS 1.3 inside the network boundary. |

## **Detailed Compliance Narratives & Audit Evidence**

### **1\. Verification of Input Validation & Sanitization (NZISM Chapter 17\)**

To satisfy NZISM requirements for input validation within an LLM landscape, we cannot rely on regex or fixed string matching due to the natural language interface of generative AI.

#### **Audit Evidence / Technical Realization:**

Microsoft Foundry's Content Safety engine acts as our digital quality inspector. When a user submits an input, it passes through **Intervention Point 1**. The runtime engine scores the prompt across four risk vectors: Hate, Sexual, Violence, and Self-Harm.  
For public sector compliance, these thresholds are locked to **Low** sensitivity. Any input generating a risk score greater than 2 is immediately terminated with a sanitized error message returned to the caller, preventing the base LLM from processing malicious or unaligned instructions.

\[User Natural Language Prompt\]   
              │  
              ▼  
┌────────────────────────────────────────────────────────┐  
│ Intervention Point 1: Content Safety Input Filters     │  
│  \- Jailbreak / Prompt Shield Evaluation                │  
│  \- Risk Scoring (Hate, Violence, Sexual, Self-Harm)   │  
└────────────────────────┬───────────────────────────────┘  
                         │  
         ┌───────────────┴───────────────┐  
         ▼ (Score \<= 2\)                  ▼ (Score \> 2\)  
┌────────────────────────────────┐ ┌────────────────────────────────┐  
│ Pass to LLM Inference Layer     │ │ Block Action & Emit Event      │  
│ (Deterministic Tokenization)   │ │ (Route to Sentinel Analytics)   │  
└────────────────────────────────┘ └────────────────────────────────┘

### **2\. Operationalizing Least Privilege for AI Agents (NZISM Chapter 14\)**

A major risk to government accreditation is the "Confused Deputy" scenario—where a low-privileged citizen or internal user uses an AI agent to execute system tasks beyond their clear authorization bounds.

#### **Audit Evidence / Technical Realization:**

We eliminate this risk by enforcing unique **Entra ID Agent IDs**. When an agent requires access to an internal repository (e.g., Spatial Data Lake or Policy Shares via Azure AI Search), it does not act under a global "System Admin" service account. Instead, it relies on its individual User-Assigned Managed Identity.  
During **Intervention Point 2 (Tool Call Validation)**, the orchestration framework verifies that the proposed data query or API command matches the precise schema allocated to that Agent ID. If a user tries to hijack the agent via prompt injection to read an unauthorized directory, the underlying Azure RBAC mechanism silently drops the token request at the identity boundary, enforcing absolute non-repudiation and isolation.

### **3\. Cloud Auditing & Monitoring Alignment (NZISM Chapter 16 & 23\)**

NZISM demands that cloud-hosted systems maintain continuous, centralized monitoring capable of supporting incident response workflows during a cyber security compromise.

#### **Audit Evidence / Technical Realization:**

Every decision made by the Microsoft Foundry guardrail mechanism (including silent programmatic text-refinement retries or outright content blocks) generates a structured payload stream. This configuration leverages native integration with the joint **Microsoft & NCSC Azure Policy Initiative**, ensuring compliance monitoring is automated and audited in real-time.  
By routing these streams to Microsoft Sentinel, the SOC maintains absolute visibility over the operational health of our AI systems. For instance, a sudden spike in MitigationAction \== "Block" logs on a specific agent instantly highlights an active adversarial probe or a poisoned internal data source, allowing automated containment protocols to freeze the compromised agent’s Managed Identity within seconds.

## **Conclusion for the Accreditation Authority**

By implementing Microsoft Foundry with this structured guardrail framework and Entra ID Agent Identity approach, the department satisfies the core security principles of the NZISM: **Confidentiality, Integrity, Availability, and Accountability.**  
The non-deterministic behavior of generative models is successfully bound by deterministic security controls, establishing a defensible, robust environment capable of managing sensitive government workflows.


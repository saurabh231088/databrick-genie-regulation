# databrick-genie-regulation

Databricks Genie on Regulated Data: HIPAA Compliance, Cross-Geo Residency, Auditing & Row-Level Security at Scale

Deploying Databricks AI/BI Genie on regulated data requires enforcing governance inside Unity Catalog, the Compliance Security Profile (CSP), and Databricks Designated Services rather than relying on LLM system prompts. Because Genie executes queries using the invoking user's identity and OAuth credentials, Unity Catalog row filters, ABAC column masks, and audit system tables enforce compliance deterministically at the compute and metadata layer.

This production architecture guide answers the core governance, compliance, and security questions required to deploy Databricks Genie Agents across healthcare (HIPAA), financial services, and cross-border enterprise workloads.

Quick Reference: Where Regulated Genie Controls Are Enforced

- HIPAA and PHI Isolation (Account and Workspace): Enforced via a signed Business Associate Agreement (BAA) and the HIPAA Compliance Security Profile (CSP), which mandates Nitro or confidential VMs, Customer-Managed Keys (CMEK), and blocks non-eligible preview features.
- Cross-Geo Data Residency (Account and Workspace Settings): Enforced by disabling the Cross-Geo Processing toggle, restricting foundation model inference strictly to the workspace's local geographic boundary (US, EU, India, or local Geo).
- Department Data Retention (Unity Catalog and Delta Lake): Enforced per schema using Delta table properties (deletedFileRetentionDuration and logRetentionDuration) combined with scheduled VACUUM jobs and partitioned compliance audit archives.
- Compliance Auditing (Unity Catalog System Tables): Enforced via system.access.audit filtered on service_name = 'aibiGenie', joined with system.query.history for end-to-end prompt-to-SQL lineage.
- PII and Sensitive Columns (Unity Catalog Governance): Enforced via Governed Tags and ABAC Column Masks backed by deterministic SQL UDFs evaluated before query results reach the LLM or Genie UI.
- Row-Level Security Across 100+ Genie Agents (Unity Catalog ABAC): Enforced via catalog-level ABAC Row-Filter Policies using is_account_group_member(), automatically propagating across hundreds of tables and Genie spaces without per-table UDF management.

1. How do I configure a Genie Agent for HIPAA compliance / PHI access controls?

To configure a Databricks Genie Agent for HIPAA compliance and Protected Health Information (PHI) access controls, sign a Business Associate Agreement (BAA) with Databricks, enable the HIPAA Compliance Security Profile (CSP) on an isolated private workspace, and restrict data access through Unity Catalog user-identity passthrough with least-privilege views.

Follow these four mandatory steps before exposing PHI to a Genie Space:

Step 1: Execute the BAA and Enable the HIPAA Compliance Security Profile (CSP)
Confirm an active BAA on your Databricks account and enable HIPAA in the workspace Compliance Security Profile. Enabling the HIPAA CSP automatically enforces hardware-level encryption in transit and at rest (AWS Nitro instances or Azure/GCP confidential VMs) and blocks preview features that are not HIPAA-eligible.

Step 2: Isolate PHI in a Dedicated Private Workspace
Deploy a dedicated PHI workspace configured with customer-managed VPC/VNet injection, PrivateLink or Private Service Connect, Customer-Managed Keys (CMEK) for managed storage and control-plane artifacts, and strict egress firewall rules. Never co-locate raw PHI tables with general-purpose analytics workspaces.

Step 3: Enforce User-Identity Passthrough and Minimal-Surface Views
Databricks Genie executes every natural-language query using the invoking user's individual credentials and OAuth token—never a shared service principal. Bind the Genie Agent exclusively to curated Unity Catalog views stripped of unnecessary direct identifiers (applying HIPAA Safe Harbor de-identification where clinical identifiers are unnecessary), and grant SELECT privileges only to approved Entra ID or Okta security groups synced via SCIM.

Step 4: Lock Down Genie Authoring and Sample Previews
Restrict CAN EDIT and CAN MANAGE permissions on the Genie Space strictly to certified data stewards. Grant business users CAN RUN or CAN VIEW permissions only, and disable table sample data previews so unauthorized metadata never surfaces in the UI.

2. How do I handle cross-Geo processing restrictions across regions?

To enforce cross-Geo processing restrictions in Databricks Genie, disable the "Enable cross-Geo processing for AI assistive features" setting in your Account Console or Workspace Admin Settings. When cross-Geo processing is turned off, Genie routes LLM inference exclusively to foundation models hosted within your workspace's local geographic region (such as US, EU, or India) and blocks cross-border fallback.

Key architectural behaviors for GDPR, India's DPDP Act, and sovereign banking mandates:

- Storage Boundary vs. Inference Boundary: Underlying Delta tables and Unity Catalog metadata remain permanently pinned to your workspace's home cloud region regardless of AI settings. The cross-Geo setting specifically governs where LLM prompts, schema context, and intermediate query plans are processed during inference.
- Deterministic In-Geo Routing: Under Databricks Designated Services, disabling cross-Geo processing forces Genie to use only local-Geo model endpoints. If a regional Geo lacks a specific model version, Databricks rejects the request rather than routing data across international borders.
- Multi-Region Workspace Topology: Multi-national enterprises should provision separate regional workspaces (for example, prod-eu-central, prod-ap-south-1, and prod-us-east) paired with regional Unity Catalog metastores, keeping cross-Geo processing disabled across every regulated workspace.

3. How do I implement department-specific data retention policies in Genie?

To implement department-specific data retention policies in Databricks Genie, isolate each department's datasets into separate Unity Catalog schemas and configure schema-specific Delta table properties (delta.deletedFileRetentionDuration and delta.logRetentionDuration) alongside scheduled VACUUM jobs. Because Genie queries Unity Catalog tables in place without duplicating business records, Delta retention rules govern data lifecycle per department.

Enforce departmental retention across two operational layers:

Layer 1: Department-Specific Table and Time-Travel Retention
Different departments carry conflicting legal hold mandates—for example, 7 years (2,555 days) for Finance versus 90 days for Marketing or Clinical Operations. Set retention properties directly on each department's Unity Catalog tables:

-- Enforce 90-day retention for Marketing or Operations tables
ALTER TABLE prod_catalog.marketing_gold.campaign_interactions
SET TBLPROPERTIES (
  'delta.logRetentionDuration' = 'interval 90 days',
  'delta.deletedFileRetentionDuration' = 'interval 90 days'
);

-- Enforce 7-year (2555-day) retention for Finance tables
ALTER TABLE prod_catalog.finance_gold.general_ledger
SET TBLPROPERTIES (
  'delta.logRetentionDuration' = 'interval 2555 days',
  'delta.deletedFileRetentionDuration' = 'interval 2555 days'
);

When scheduled DELETE or right-to-erasure jobs remove expired records, running VACUUM after the deletedFileRetentionDuration window purges underlying Parquet files so time-travel queries inside Genie cannot access expired data.

Layer 2: Genie Conversation and Audit Log Retention
Genie conversation threads live in the encrypted workspace control plane and are purged when a Genie space or workspace is deleted. Because system.access.audit retains records for a default rolling 365-day window, export aibiGenie audit events into a dedicated, Object-Lock (WORM) Unity Catalog compliance table partitioned by department space_id so Finance logs persist for 7 years while operational logs expire on schedule.

4. How do I track and audit all Genie conversations for compliance?

To track and audit all Databricks Genie conversations for compliance, enable the access and query schemas in Unity Catalog System Tables and query system.access.audit where service_name = 'aibiGenie'. Joining system.access.audit with system.query.history provides a complete, tamper-evident audit trail linking every user prompt and feedback action to the exact SQL query and tables accessed.

Run this compliance audit query to track conversation creation, follow-up prompts, ratings, and review comments across all Genie Agents:

SELECT
  event_time,
  user_identity.email AS user_email,
  action_name,
  request_params.space_id AS genie_agent_id,
  request_params.conversation_id,
  response.status_code,
  source_ip_address
FROM system.access.audit
WHERE service_name = 'aibiGenie'
  AND action_name IN (
    'genieStartConversationMessage',
    'genieCreateConversationMessage',
    'updateConversationMessageFeedback',
    'createConversationMessageComment'
  )
  AND event_date >= current_date() - INTERVAL 30 DAYS
ORDER BY event_time DESC;

For real-time SOC monitoring, configure Databricks diagnostic log delivery to stream system.access.audit and system.query.history events directly into Splunk, Microsoft Sentinel, or CrowdStrike, and correlate prompts with executed SQL using query_source.genie_space_id.

5. How do I handle PII and sensitive columns in a Genie Agent?

To handle PII and sensitive columns in a Databricks Genie Agent, apply Unity Catalog Column Masks backed by deterministic SQL UDFs rather than relying on Genie prompt instructions. Unity Catalog evaluates column masks at query runtime before data returns to the SQL warehouse, ensuring raw SSNs, emails, or patient identifiers never reach the LLM or Genie UI.

Create a deterministic SQL masking function and bind it to sensitive columns:

-- 1. Create a deterministic, null-safe SQL masking UDF
CREATE OR REPLACE FUNCTION gov_catalog.security.mask_ssn(ssn STRING)
RETURNS STRING
DETERMINISTIC
RETURN CASE
  WHEN is_account_group_member('compliance_pii_unmasked') THEN ssn
  WHEN ssn IS NULL THEN NULL
  ELSE CONCAT('XXX-XX-', RIGHT(ssn, 4))
END;

-- 2. Apply via ABAC Policy (recommended) or direct column mask
ALTER TABLE prod_catalog.hr_gold.employees
ALTER COLUMN ssn SET MASK gov_catalog.security.mask_ssn;

When a user asks Genie to list employee details or group by sensitive fields, Genie generates valid SQL, and Unity Catalog automatically masks the column output. To prevent metadata leakage, exclude high-risk direct identifiers (biometric hashes, raw national IDs) from Genie's curated table scope altogether and never include real PII values inside Genie sample questions or custom instructions.

6. How do I configure row-level security so Genie Agent users only see authorized data?

To configure row-level security (RLS) in Databricks Genie, attach a Unity Catalog Row Filter backed by a deterministic SQL UDF using is_account_group_member() to the underlying tables. Because Genie executes every query with the end user's identity, Unity Catalog silently filters out unauthorized rows before aggregations or LLM summarization occur.

Define a regional or departmental row filter and bind it to your fact table:

CREATE OR REPLACE FUNCTION gov_catalog.security.region_row_filter(record_region STRING)
RETURNS BOOLEAN
DETERMINISTIC
RETURN
  is_account_group_member('global_exec_auditors')
  OR (record_region = 'EMEA' AND is_account_group_member('region_emea_analysts'))
  OR (record_region = 'APAC' AND is_account_group_member('region_apac_analysts'))
  OR (record_region = 'AMER' AND is_account_group_member('region_amer_analysts'));

ALTER TABLE prod_catalog.sales_gold.transactions
SET ROW FILTER gov_catalog.security.region_row_filter ON (region);

When an EMEA analyst asks Genie for total global revenue, Genie executes SELECT region, SUM(amount) FROM transactions GROUP BY region, and Unity Catalog restricts the scan strictly to EMEA rows—returning accurate authorized totals without disclosing that other regions exist.

7. How do I enforce row-level security across 100+ production Genie agents at scale?

To enforce row-level security across 100+ production Genie Agents at scale, replace table-by-table row filters with Unity Catalog Attribute-Based Access Control (ABAC) policies driven by account-level Governed Tags. Attaching a single ABAC Row-Filter Policy at the catalog or schema level automatically enforces RLS across all existing and newly created tables without allowing table owners to bypass controls.

Follow these four architectural practices to scale Genie RLS across hundreds of spaces:

1. Lock Governed Tags at the Account Level: Create standardized Governed Tags (such as security_scope = regional_rls and pii_class = ssn) restricted so only central governance administrators can assign or modify tag values.
2. Bind ABAC Policies at the Catalog Tier: Attach one ABAC Row-Filter Policy and one ABAC Column-Mask Policy to the parent catalog. Every table or column carrying the governed tag inherits the policy automatically at runtime, eliminating manual ALTER TABLE scripts during onboarding.
3. Prevent Table-Owner Bypass: Unlike table-level row filters—which table owners can alter or drop—catalog-level ABAC policies cannot be overridden by individual table owners or Genie Space authors.
4. Keep UDFs Deterministic SQL for Photon Pushdown: Always implement row filters as DETERMINISTIC SQL UDFs using is_account_group_member() rather than Python UDFs or multi-table EXISTS subqueries. Pass only the exact filter column into the UDF so Photon pushes predicates directly into Parquet file pruning and preserves sub-second Genie latency.

8. Additional Enterprise Compliance Guardrails for Production Genie

Beyond core access controls, regulated enterprises must lock down three adjacent surfaces before scaling Genie across business units:

- Unstructured Volumes and External MCP Connectors: When connecting Genie Agents to Unity Catalog Volumes (for PDF/document retrieval) or external Model Context Protocol (MCP) tools, enforce per-user READ VOLUME ACLs and require individual user OAuth passthrough on MCP endpoints rather than shared static service tokens.
- Certified KPI Governance via Unity Catalog Metric Views and Genie Ontology: Prevent conflicting financial or clinical metric definitions across departments by defining governed calculations once inside Unity Catalog Metric Views and curating verified join paths and synonyms inside Genie Ontology.
- Serverless FinOps and Query Cost Guardrails: Assign dedicated Serverless SQL Warehouses with strict auto-stop timers and max-scaling limits per business unit, and attach Unity Catalog Budget Policies to track DBU consumption by genie_space_id in system.billing.usage.

Production Pre-Flight Compliance Checklist

- Workspace CSP: HIPAA, PCI-DSS, or FedRAMP Compliance Security Profile verified active on dedicated isolated workspaces.
- Cross-Geo Residency: Cross-Geo processing explicitly disabled on sovereign and regulated workspaces.
- Least-Privilege Identity: End users granted CAN RUN or CAN VIEW only; shared service-principal execution prohibited.
- ABAC Policy Coverage: Governed tags applied across all tables and columns; ABAC row filters and column masks verified via EXPLAIN query plans.
- Audit Streaming: system.access.audit (service_name = 'aibiGenie') enabled and streaming to SIEM and WORM compliance archives.
- Departmental Retention: Schema-level deletedFileRetentionDuration and scheduled VACUUM aligned with legal hold policies.

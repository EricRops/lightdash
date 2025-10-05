Our goal is to make enhancements to this repository to enable a Multi-Tenant architecture with Lightdash and BigQuery, while, critically, ensuring tenant data isolation with no chance of data leaking between tenants.

**For overall context, here is the final architecture we are stiving for:**

Architecture Component Breakdown
1. Project Structure
The architecture uses 5 distinct project types:

Data Pipeline Project: Receives, processes, and distributes all tenant data. Contains Dataflow jobs that dual-write incoming data to both internal analytics datasets and tenant-specific datasets.
Combined Tenant Data Project: The core project containing one dataset per customer (your 10,000 tenant datasets would live here). This is strictly a storage layer—queries don't execute here.
Internal Development Projects: For your analytics teams to build features and evaluate tenant data using the combined cross-tenant dataset.
End-User Application Projects: Where Lightdash would be deployed. These projects contain "resources designed to interact with end users" and use tenant-scoped service accounts to access tenant datasets.
Reservation Compute Tier Projects: Separate projects (Low Tier, High Tier, etc.) where actual BigQuery queries execute. Each tier has its own slot reservation, and tenants are mapped to tiers based on their service level.

2. Data Flow Architecture
The key innovation is dual-writing with fully written tables:

Dataflow ingests data once
Writes to internal cross-tenant analytics dataset (for your internal BI)
Simultaneously writes to each tenant's individual dataset (tenant_001_data, tenant_002_data, etc.)
Uses "fully written tables instead of authorized views" to avoid the authorized view limit of ~2,500 per dataset

3. Compute Separation via Reservation Tiers
Queries don't execute in the data storage project. Instead:

Lightdash queries would originate from an "End-User Application Project"
Queries are routed to a "Compute Tier Project" (Low, High, or dedicated)
Each compute tier project is assigned to a BigQuery reservation with allocated slots
Fair scheduling distributes slots within each tier, and unused slots automatically flow between tiers

How Data Leakage is Prevented
This architecture implements defense-in-depth with multiple isolation layers:
Layer 1: Tenant-Scoped Service Accounts
The GCP doc explicitly states: "We recommend that you use tenant-scoped service accounts to access tenant datasets."
This means:

Each tenant (or tenant tier) has a dedicated service account
Service account tenant-001-sa@project.iam.gserviceaccount.com has BigQuery Data Viewer permission ONLY on project.tenant_001_data dataset
When a user from Tenant 001 queries via Lightdash, the application impersonates tenant-001-sa to execute the query
BigQuery IAM enforces that this service account can only read from tenant_001_data—any attempt to query tenant_002_data is rejected at the infrastructure level

Layer 2: VPC Service Controls Perimeters
Creates security boundaries around projects:

Tenant Data Perimeter: Wraps the Combined Tenant Data Project and Compute Tier Projects. "Enforce all services to prevent access from outside of the organization."
External Applications Perimeter: Wraps the Lightdash deployment project
Perimeter Bridge: Connects External Applications ↔ Tenant Data, allowing only authorized query traffic

This prevents:

Data exfiltration to external services
Unauthorized project-to-project access
Accidental data export outside the organization

Layer 3: Physical Dataset Separation
Each tenant's data lives in a completely separate BigQuery dataset with:

Independent IAM policies
Separate encryption keys (optionally CMEK per tenant)
Isolated metadata and schemas
No shared storage resources

Even if application logic fails, the datasets are physically isolated—there's no shared table that could leak data via a WHERE clause bug.
Layer 4: Query Execution Project Scoping
Queries execute in Compute Tier Projects, not the data storage project. The query execution context includes:

Which service account initiated the query (mapped to tenant)
Which datasets that service account can access (enforced by IAM)
Audit logs capturing tenant_id, query text, and accessed datasets

Layer 5: Audit Logging
Every query is logged with:

Service account used (maps to tenant)
Datasets accessed
Rows returned and bytes scanned
Originating project and user

Automated monitoring can alert on anomalies like:

Service account accessing unexpected datasets
Cross-tenant queries (should be impossible, but triggers investigation)
Unusual data volume access patterns

Data Leakage Prevention Flow
When a Tenant 001 user queries via Lightdash:

Authentication: User authenticates via Keycloak, JWT contains tenant_id: tenant_001
Service Account Selection: Lightdash maps tenant_001 to tenant-001-sa service account
Impersonation: Lightdash impersonates tenant-001-sa to obtain short-lived credentials
Query Submission: Query is submitted to BigQuery from End-User Application Project, using tenant-001-sa credentials
IAM Enforcement: BigQuery checks IAM—tenant-001-sa can only access tenant_001_data dataset
Execution: Query executes in Compute Tier Project, reading only from tenant_001_data
Results Return: Results flow back through VPC Service Controls perimeter bridge to Lightdash
Audit Log: Entry created with tenant_id, service account, datasets accessed, query text

If the application tries to query tenant_002_data:

BigQuery IAM immediately rejects with "Access Denied"
Attempt is logged in audit logs
No data is returned

Compatibility with Your Lightdash Requirements
Let me assess against your original criteria:
✅ Multiple tenants with identical schemas across separate datasets: Perfect match. This is exactly what the architecture is designed for—the doc explicitly recommends "dataset-per-tenant design" at scale.
✅ Single Lightdash instance serving all tenants: Yes. Lightdash would run in the "End-User Application Project" and serve all 10,000 tenants.
✅ Query routing to correct dataset based on user attributes: Yes, via service account impersonation based on the authenticated user's tenant_id from Keycloak.
✅ Data isolation enforced at database/infrastructure level, NOT application layer: This is the key strength. The architecture uses BigQuery IAM + VPC Service Controls for infrastructure-level enforcement. Even if Lightdash has bugs, BigQuery prevents cross-tenant access.
✅ 100-200 concurrent queries total: Perfect fit. The doc notes "BigQuery concurrency has a default of 100 queries per project that issues queries" and this can be increased. With compute tier projects, you can easily handle 100-200 queries.
Implementation Strategy for Lightdash
Here's how you'd implement this with Lightdash:
1. Create 10,000 Tenant Service Accounts
bash# Create service accounts for each tenant
for i in {1..10000}; do
  tenant_id=$(printf "tenant_%05d" $i)
  sa_name="${tenant_id}-sa"
  
  gcloud iam service-accounts create $sa_name \
    --display-name="Service account for $tenant_id" \
    --project=combined-tenant-data-project
  
  # Grant access only to this tenant's dataset
  gcloud projects add-iam-policy-binding combined-tenant-data-project \
    --member="serviceAccount:${sa_name}@combined-tenant-data-project.iam.gserviceaccount.com" \
    --role="roles/bigquery.dataViewer" \
    --condition="resource.name.startsWith('projects/combined-tenant-data-project/datasets/${tenant_id}_data')"
done
2. Grant Lightdash Service Account Impersonation Rights
bash# Lightdash's main service account
LIGHTDASH_SA="lightdash-main@end-user-app-project.iam.gserviceaccount.com"

# Grant token creator role on all tenant SAs
for i in {1..10000}; do
  tenant_sa="tenant_$(printf "%05d" $i)-sa@combined-tenant-data-project.iam.gserviceaccount.com"
  
  gcloud iam service-accounts add-iam-policy-binding $tenant_sa \
    --member="serviceAccount:${LIGHTDASH_SA}" \
    --role="roles/iam.serviceAccountTokenCreator" \
    --project=combined-tenant-data-project
done
3. Modify Lightdash to Use Service Account Impersonation
typescript// In Lightdash's BigQuery warehouse adapter
import { GoogleAuth } from 'google-auth-library';
import { BigQuery } from '@google-cloud/bigquery';

async function getBigQueryClientForTenant(tenantId: string): Promise<BigQuery> {
  // Get tenant's service account email
  const tenantSA = `tenant_${tenantId}-sa@combined-tenant-data-project.iam.gserviceaccount.com`;
  
  // Create impersonated credentials
  const auth = new GoogleAuth({
    scopes: ['https://www.googleapis.com/auth/bigquery']
  });
  
  const client = await auth.getClient();
  const impersonatedClient = new ImpersonatedCredentials({
    sourceClient: client,
    targetPrincipal: tenantSA,
    targetScopes: ['https://www.googleapis.com/auth/bigquery'],
    lifetime: 3600 // 1 hour
  });
  
  // Create BigQuery client with impersonated credentials
  return new BigQuery({
    authClient: impersonatedClient,
    projectId: 'compute-tier-standard', // Where queries execute
    // No need to specify dataset—queries will explicitly reference it
  });
}

// When user queries
async function executeQuery(sql: string, user: User) {
  const tenantId = user.attributes.tenant_id; // From Keycloak
  const bqClient = await getBigQueryClientForTenant(tenantId);
  
  // Rewrite SQL to reference tenant dataset
  const tenantDataset = `tenant_${tenantId}_data`;
  const rewrittenSql = rewriteDatasetReferences(sql, tenantDataset);
  
  // Execute query - IAM will enforce access
  const [rows] = await bqClient.query({
    query: rewrittenSql,
    location: 'US'
  });
  
  return rows;
}
4. Configure VPC Service Controls
Create perimeters as described in the GCP doc, ensuring:

Lightdash's End-User Application Project is in "External Applications" perimeter
Combined Tenant Data Project is in "Tenant Data" perimeter
Bridge connects them for query traffic only

Key Advantages of This Approach

No Proxy Needed: Unlike the hybrid approach I recommended earlier, this uses GCP-native service account impersonation. No custom proxy to maintain.
Infrastructure-Level Isolation: BigQuery IAM + VPC Service Controls provide bulletproof isolation. Even if Lightdash is completely compromised, attackers can't access other tenants' data.
Google-Validated Pattern: This is Google's explicitly recommended architecture for "tens of thousands" of tenants. You're following a proven pattern.
No SQL Rewriting Complexity: The SQL rewriting is minimal—just inject the tenant dataset name. No complex regex parsing of JOINs, CTEs, etc.
Audit Trail: Complete audit trail of which service account (tenant) accessed which data, when, and how much.
Scalability: Handles 10,000 tenants without architectural changes. Google confirms this scales.

Potential Challenges

Service Account Management: 10,000 service accounts is operationally complex. You'll need automation for provisioning, IAM binding, and lifecycle management.
GCP Quotas: Default soft limit of 100 service accounts per project. You'd need to request quota increases or spread across multiple projects (100 projects × 100 SAs each).
Token Caching: Impersonated tokens expire after 1 hour. Implement caching in Lightdash to avoid re-impersonating on every query.
Cost: Service account impersonation is free, but managing 10,000 IAM bindings adds operational overhead.

My Revised Recommendation
This GCP architecture is superior to the proxy-based approach I initially recommended for your use case because:

Stronger isolation: Infrastructure-level enforcement vs application-level
Less code: Uses Google's IAM instead of custom routing logic
Lower operational complexity: No proxy infrastructure to manage
Proven at scale: Google explicitly validates this for "tens of thousands" of tenants

**Assumptions**:
* Each user will be assigned a "tenant_id" user attribute upon user creation. For example, a user from Levi's will have a user attribute 
like {"tenant_id": "levi"}. The tenant id is associated with the BigQuery dataset they can access (ex: `combined-tenant-data-project.levi_data`),
as well as the tenant service account that only gives access to that dataset (ex: `levi-sa@combined-tenant-data-project.iam.gserviceaccount.com`)
* 

**Here are the enhancements we need to make to this repository to support this architecture using Lightdash:**
  1. Whenever Lightdash sends a query to BigQuery, inject the tenant dataset name into the SQL query. Throw an error if there is no tenant ID.
  2. Lightdash must impersonate the tenant GCP service account based on the tenant_id. For example, when `tenant_id` is "levi"
  lightdash must impersonate the levi-sa@combined-tenant-data-project.iam.gserviceaccount.com so that BigQuery IAM enforces that the query
  can only read from the combined-tenant-data-project.levi_data dataset



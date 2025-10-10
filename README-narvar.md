## Commands needed for Impersonation

### Give lightdash-poc-sa permissions to impersonate the tenant SAs
```
gcloud iam service-accounts add-iam-policy-binding \
    levi-sa@narvar-qa-202121.iam.gserviceaccount.com \
    --member="serviceAccount:lightdash-poc-sa@narvar-qa-202121.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountTokenCreator" \
    --project=narvar-qa-202121
```

### Show all the SAs the lightdash has permission to impersonate
```
gcloud asset search-all-iam-policies \
    --scope=projects/narvar-qa-202121 \
    --query="policy:lightdash-poc-sa@narvar-qa-202121.iam.gserviceaccount.com AND policy:roles/iam.serviceAccountTokenCreator"
```

### Confirm permissions of a dataset
bq show --format=prettyjson narvar-qa-202121:lightdash_test_combined | grep -A 20 '"access"' | head -30


### Grant permissions to run jobs for a specific SA
gcloud projects add-iam-policy-binding narvar-qa-202121 \
    --member="serviceAccount:levi-sa@narvar-qa-202121.iam.gserviceaccount.com" \
    --role="roles/bigquery.jobUser" \
    --condition=None

### Grant SA viewer permissions for a specific dataset
gcloud projects add-iam-policy-binding narvar-qa-202121 \
    --member="serviceAccount:levi-sa@narvar-qa-202121.iam.gserviceaccount.com" \
    --role="roles/bigquery.dataViewer" \
    --condition='expression=resource.name.startsWith("projects/narvar-qa-202121/datasets/lightdash_test_levi"),title=Access to lightdash_test_levi'


### List permissions for a given SA:
gcloud projects get-iam-policy narvar-qa-202121 \
    --flatten="bindings[].members" \
    --filter="bindings.members:serviceAccount:levi-sa@narvar-qa-202121.iam.gserviceaccount.com" \
    --format="table(bindings.role, bindings.members)"

### List all datasets that a SA has access to (project level and dataset level)
./../../Narvar/Project\ Documents/Metabase\ Replacement/scripts/list_sa_datasets_v2.sh


## DEMO:
- First callout the data visibility restriction in the lightdash UI
- Show the user account attributes 
- Show narvar user querying combined data (has permission).  Show the impersonated SA in bigquery
- Show narvar user querying belk data (gets SA permission error)
- Show levi user querying levi data. Show the impersonated SA in bigquery
- Show levi user querying combined and belk data (gets SA permission error). 
   - Mention that this would only happen due to some bug! We will normally only show levi tables to the levi user


## TODO:
- SQL query discrepancy (actually, if the impersonation logic from the Demo is accepted, we can drop the SQL injection altogether)
- desired behavior:  "narvar" users impersonate the "narvar-sa" SA,  BUT NO SQL dataset rewrite
- tenant users:  impersonates the  "<tenant>-sa" , auto SQL dataset rewrite

- Lightdash user data table visibility based on tenant_id (DBT config)

- lightdash is sending duplicate query jobs to BQ!! If I load one chart then it sends one job. Good.
  - But, when I load a 2nd chart, it sends 1 new job to BQ for the new chart, PLUS 3 or 4 identical jobs from the chart I was at previously!

- no tenant_id = No queries allowed

- Remove the unnecessary debug logs

## SQL discrepancy root cause:
Critical Security Issue: SQL Injection Bypass

  Location of the bypass: packages/backend/src/utils/QueryBuilder/MetricQueryBuilder.ts:306 and :381

  The dimension and metric compiledSql fields are used directly in SELECT clauses:

  // Line 306 - Dimensions
  selects[id] = `  ${dimension.compiledSql} AS ${quotedAlias}`;

  // Line 381 - Metrics  
  return `  ${metric.compiledSql} AS ${fieldQuoteChar}${alias}${fieldQuoteChar}`;

  The Problem:

  1. Dimensions and metrics are compiled EARLY (during explore compilation in exploreCompiler.ts:574-612)
  2. Their compiledSql contains table references like  `v_return_details_levi`.return_process_info_id
  3. injectTenantDataset() only runs on:
    - Base table FROM clause (line 745)
    - JOIN clauses (line 812)
  4. But NOT on the dimension/metric compiledSql fields

  What happens:

  The compiledSql is generated from the ${TABLE} reference during explore compilation. At line 721-726 in exploreCompiler.ts:

  if (ref === 'TABLE') {
      const fieldQuoteChar = this.warehouseClient.getFieldQuoteChar();
      return {
          sql: `${fieldQuoteChar}${currentTable}${fieldQuoteChar}`,
          tablesReferences: new Set([currentTable]),
      };
  }

  This creates something like `v_return_details_levi` which gets used in the dimension SQL like  `v_return_details_levi`.column_name. This gets cached in the explore and reused for all queries.

  Then when injectTenantDataset() transforms the FROM/JOIN clauses to use the tenant dataset, the dimension SQL still references the original table alias which BigQuery then resolves using the original 
  database.schema from the dbt model.

  The Result:
  - UI shows: FROM \narvar-qa-202121.lightdash_test_combined.v_return_details` ✅
  - BigQuery receives: SELECT ... FROM \narvar-qa-202121.lightdash_test_levi.v_return_details` ❌

  This is a critical security vulnerability - users see one query but a completely different query executes.

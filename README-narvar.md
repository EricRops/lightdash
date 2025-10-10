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



## TODO:

- Lightdash user data table visibility based on tenant_id (DBT config)

- Embedding POC

- lightdash is sending DUPLICATE query jobs to BQ!! If I load one chart then it sends one job. Good.
  - But, when I load a 2nd chart, it sends 1 new job to BQ for the new chart, PLUS 3 or 4 identical jobs from the chart I was at previously!

- Remove the unnecessary debug logs
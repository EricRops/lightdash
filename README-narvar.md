## Commands needed for Impersonation

- Give lightdash SA permissions to impersonate the tenant SAs
```
gcloud iam service-accounts add-iam-policy-binding \
    narvar-sa@narvar-qa-202121.iam.gserviceaccount.com \
    --member="serviceAccount:lightdash-poc-sa@narvar-qa-202121.iam.gserviceaccount.com" \
    --role="roles/iam.serviceAccountTokenCreator" \
    --project=narvar-qa-202121
```

- Show all the SAs the lightdash has permission to impersonate
```
gcloud asset search-all-iam-policies \
    --scope=projects/narvar-qa-202121 \
    --query="policy:lightdash-poc-sa@narvar-qa-202121.iam.gserviceaccount.com AND policy:roles/iam.serviceAccountTokenCreator"
```

- Confirm permissions of a dataset
bq show --format=prettyjson narvar-qa-202121:lightdash_test_combined | grep -A 20 '"access"' | head -30

TODO:
fix linting errors
# gcp-aggressor
Whitelist egress traffic to dynamic hostname targets in GCP


## HOWTO

### Setting firewall rules
* Create a deny-all firewall rule 
```
gcloud compute --project=$PROJECT_ID firewall-rules create agressor-deny-all --direction=EGRESS --priority=1000 --network=$NETWORK_NAME --action=DENY --rules=all --destination-ranges=0.0.0.0/0
```
* Create a stub firewall rule to whitelist access to google.com:
```
gcloud compute --project=$PROJECT_ID firewall-rules create aggressor-google-com --direction=EGRESS --priority=250 --network=$NETWORK_NAME --action=ALLOW --rules=all --destination-ranges=0.0.0.0/0
```

### Setting the GCP workflow
* Create a service account for aggressor GCP workflow:
```
gcloud iam service-accounts create agressor
```
* Grant this service account log write permissions:
```
gcloud projects add-iam-policy-binding $PROJECT_ID --member "serviceAccount:agressor@$PROJECT_ID.iam.gserviceaccount.com" --role "roles/logging.logWriter"
```
* Create the aggressor GCP workflow:
```
gcloud workflows deploy aggressor \
--source=aggressor.yaml \
--service-account=agressor@yarel-playground.iam.gserviceaccount.com
```

### Setting the Cloud Scheduler job
* Create a service account:
```
gcloud iam service-accounts create aggressor-scheduler
```
* Assign workflow invoker permissions to the service account:
```
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:aggressor-scheduler@$PROJECT_ID.iam.gserviceaccount.com \
  --role roles/workflows.invoker
```
* Create a Cloud Scheduler job:
```
gcloud scheduler jobs create http aggressor \
  --schedule="*/5 * * * *" \
  --uri="https://workflowexecutions.googleapis.com/v1/projects/$PROJECT_ID/locations/$REGION_NAME/workflows/aggressor/executions" \
  --message-body="{\"argument\": \"{\\n    \\\"hostname\\\": \\\"google.com\\\",\\n    \\\"ruleName\\\": \\\"aggressor-google-com\\\"\\n}\"}" \
  --time-zone="Etc/UTC" \
  --oauth-service-account-email="aggressor-scheduler@$PROJECT_ID.iam.gserviceaccount.com"
```



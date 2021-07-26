# gcp-aggressor
Whitelist egress traffic to dynamic hostname targets in GCP.

As an important least privelege security principle, if you know different components in your GCP projects only communicate towards a handful of domain names, you may want to whitelist traffic towards these domains, and block all other traffic to the internet.

GCP Aggressor will periodically resolve your requested domain names A records using Google DNS, and will automatically configure GCP firewall rules to only allow traffic to these IPs.

Important Note: It will only work with domains which returns consistent DNS A records for different clients. For example, if the DNS server is configured to return results based on geoproximity, the aggressor might get different results than your client, if it is running in different regions, for example.


# HOWTO

## Global one-time setup

### Setting up the GCP Workflow
* Create a service account for aggressor GCP workflow:
```
gcloud iam service-accounts create aggressor
```
* Grant this service account log write permissions:
```
gcloud projects add-iam-policy-binding $PROJECT_ID --member "serviceAccount:aggressor@$PROJECT_ID.iam.gserviceaccount.com" --role "roles/logging.logWriter" 
```
* Grant this service account security admin permissions:
```
gcloud projects add-iam-policy-binding $PROJECT_ID --member "serviceAccount:aggressor@$PROJECT_ID.iam.gserviceaccount.com" --role "roles/compute.securityAdmin"
```
* Create the aggressor GCP workflow:
```
gcloud workflows deploy aggressor \
--source=aggressor.yaml \
--service-account=aggressor@yarel-playground.iam.gserviceaccount.com
```

### Setting up the global deny-all firewall rule
* Create a deny-all firewall rule 
```
gcloud compute --project=$PROJECT_ID firewall-rules create aggressor-deny-all --direction=EGRESS --priority=1000 --network=$NETWORK_NAME --action=DENY --rules=all --destination-ranges=0.0.0.0/0
```

### Setting up service account for cloud scheduler

* Create a service account for the scheduler jobs:
```
gcloud iam service-accounts create aggressor-scheduler
```
* Assign workflow invoker permissions to the service account:
```
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member serviceAccount:aggressor-scheduler@$PROJECT_ID.iam.gserviceaccount.com \
  --role roles/workflows.invoker
```

# Per-hostname Setup

As an example we will whitelist `google.com`. This can be done to any hostname, and should be repeated per hostname.

### Setting firewall rules

* Create a stub firewall rule to whitelist access to google.com (with a fake CIDR range which will get modified by the workflow):
```
gcloud compute --project=$PROJECT_ID firewall-rules create aggressor-google-com --direction=EGRESS --priority=250 --network=$NETWORK_NAME --action=ALLOW --rules=all --destination-ranges=1.1.1.1/32
```

### Setting the Cloud Scheduler job

* Create a Cloud Scheduler job for this hostname:
```
gcloud scheduler jobs create http aggressor-google-com \
  --schedule="*/5 * * * *" \
  --uri="https://workflowexecutions.googleapis.com/v1/projects/$PROJECT_ID/locations/$REGION_NAME/workflows/aggressor/executions" \
  --message-body="{\"argument\": \"{\\n    \\\"hostname\\\": \\\"google.com\\\",\\n    \\\"ruleName\\\": \\\"aggressor-google-com\\\"\\n}\"}" \
  --time-zone="Etc/UTC" \
  --oauth-service-account-email="aggressor-scheduler@$PROJECT_ID.iam.gserviceaccount.com"
```



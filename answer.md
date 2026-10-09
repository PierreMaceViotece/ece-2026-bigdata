# Lab 2: Object storage with S3, answers

## 1. Why are the credentials stored in a Secret and not in the ConfigMap?

A Secret is meant for sensitive data. Access to it can be restricted separately with RBAC, its values are not displayed in clear text by default and it can be encrypted at rest. A ConfigMap is plain text, readable by anyone who can read the configuration of the namespace, and it is easily leaked in logs or committed to Git by mistake.

## 2. What happens if the Job is executed again tomorrow? How would a production platform provide credentials to a Job?

The Onyxia credentials are temporary, so tomorrow the Job would fail with an `ExpiredToken` or `AccessDenied` error. A production platform would not copy keys into the cluster. It would give the Job its own identity, for example a Kubernetes service account mapped to an IAM role, or it would fetch short-lived credentials from a secret manager such as Vault, rotated automatically.

## 3. How would you turn this Job into a daily ingestion?

Replace the Job with a Kubernetes CronJob using the same pod template and a schedule such as `0 2 * * *` (every day at 2am). Another option is to trigger the Job from an orchestrator such as Airflow, which also handles retries, dependencies and monitoring.

## Note

The section "Upload (PUT)" could not be completed: `aws s3 cp` returned `InternalError when calling the PutObject operation`, and uploading a file from the Onyxia File explorer failed as well, which points to an issue with the S3 storage of the platform.

# AWS Backup Configuration – Standalone RDS

## 1. Objective

Configure AWS Backup for the standalone RDS database with the following retention policy:

| Backup Type            | Frequency       | Retention |
| ---------------------- | --------------- | --------: |
| Daily/Hourly snapshots | 3 times per day |    7 days |
| Weekly snapshot        | 1 per week      |   30 days |
| Monthly snapshot       | 1 per month     |  365 days |

### Schedule

The configuration below uses UTC:

* Daily: `00:00`, `08:00`, `16:00 UTC`
* Weekly: Every Sunday at `01:00 UTC`
* Monthly: 1st day of every month at `02:00 UTC`

> **Important:** AWS Backup cron schedules use UTC.

---

# 2. AWS CloudShell Authentication

AWS CloudShell automatically uses the IAM identity associated with the current AWS Console session.

You do **not** need to run:

```text
aws configure
```

and you should **not** enter an access key or secret key.

Verify the current identity:

```bash
aws sts get-caller-identity
```

Example:

```json
{
    "UserId": "AIDAXXXXXXXXXXXXX",
    "Account": "123456789012",
    "Arn": "arn:aws:iam::123456789012:user/example"
}
```

If the console session is using an assumed role, the ARN may look like:

```text
arn:aws:sts::123456789012:assumed-role/RoleName/session-name
```

---

# 3. Set AWS Region

Set the region where the RDS database exists.

Example for Mumbai:

```bash
export AWS_REGION=ap-south-1
```

Verify:

```bash
aws configure get region
```

If CloudShell does not have the region configured, use:

```bash
aws configure set region "$AWS_REGION"
```

Verify:

```bash
aws sts get-caller-identity
```

---

# 4. Find the RDS Database

List all RDS instances:

```bash
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,Engine,DBInstanceStatus,DBInstanceArn]' \
  --output table
```

Example:

```text
--------------------------------------------------------------
|                  DescribeDBInstances                       |
+--------------------+---------+-----------+-----------------+
| clientportal-prod  | mysql   | available | arn:aws:rds... |
| orders-prod        | mysql   | available | arn:aws:rds... |
+--------------------+---------+-----------+-----------------+
```

Set the RDS instance that needs backup:

```bash
export RDS_INSTANCE_ID="YOUR-RDS-INSTANCE-ID"
```

Example:

```bash
export RDS_INSTANCE_ID="clientportal-prod"
```

Get the ARN:

```bash
export RDS_ARN=$(aws rds describe-db-instances \
  --db-instance-identifier "$RDS_INSTANCE_ID" \
  --query 'DBInstances[0].DBInstanceArn' \
  --output text)
```

Verify:

```bash
echo "$RDS_ARN"
```

---

# 5. Tag the RDS Database

We will use a tag to identify which RDS resources should be protected by this backup plan.

Add:

```text
Backup=true
```

Run:

```bash
aws rds add-tags-to-resource \
  --resource-name "$RDS_ARN" \
  --tags Key=Backup,Value=true
```

Verify:

```bash
aws rds list-tags-for-resource \
  --resource-name "$RDS_ARN" \
  --output table
```

Confirm that you see:

```text
Backup    true
```

> **Important:** If multiple RDS instances have `Backup=true`, the backup plan will select all of them.

---

# 6. Check AWS Backup IAM Role

Check whether the standard AWS Backup service role already exists:

```bash
aws iam get-role \
  --role-name AWSBackupDefaultServiceRole
```

If it exists, **do not create another role**.

Check attached policies:

```bash
aws iam list-attached-role-policies \
  --role-name AWSBackupDefaultServiceRole \
  --output table
```

You should have the AWS Backup service policies attached.

Expected policies include:

```text
AWSBackupServiceRolePolicyForBackup
AWSBackupServiceRolePolicyForRestores
```

---

# 7. Create AWS Backup Service Role if Required

Only perform this step if the role does not exist.

Create the role:

```bash
aws iam create-role \
  --role-name AWSBackupDefaultServiceRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": {
          "Service": "backup.amazonaws.com"
        },
        "Action": "sts:AssumeRole"
      }
    ]
  }'
```

Attach the backup policy:

```bash
aws iam attach-role-policy \
  --role-name AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForBackup
```

Attach the restore policy:

```bash
aws iam attach-role-policy \
  --role-name AWSBackupDefaultServiceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSBackupServiceRolePolicyForRestores
```

Verify:

```bash
aws iam list-attached-role-policies \
  --role-name AWSBackupDefaultServiceRole \
  --output table
```

---

# 8. Get AWS Account ID

```bash
export ACCOUNT_ID=$(aws sts get-caller-identity \
  --query Account \
  --output text)
```

Verify:

```bash
echo "$ACCOUNT_ID"
```

Set the backup role ARN:

```bash
export BACKUP_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/AWSBackupDefaultServiceRole"
```

Verify:

```bash
echo "$BACKUP_ROLE_ARN"
```

---

# 9. Create Backup Vault

Create a dedicated vault:

```bash
export BACKUP_VAULT="Standalone-RDS-Vault"
```

Create:

```bash
aws backup create-backup-vault \
  --backup-vault-name "$BACKUP_VAULT"
```

If the vault already exists, AWS will return an error indicating that it already exists. In that case, continue.

Verify:

```bash
aws backup list-backup-vaults \
  --output table
```

---

# 10. Create Backup Plan JSON

Create the backup plan definition:

```bash
cat > backup-plan.json <<'EOF'
{
  "BackupPlan": {
    "BackupPlanName": "Standalone-RDS-Backup-Plan",
    "Rules": [
      {
        "RuleName": "RDS-3xDaily-7Days",
        "TargetBackupVaultName": "Standalone-RDS-Vault",
        "ScheduleExpression": "cron(0 0,8,16 * * ? *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 180,
        "Lifecycle": {
          "DeleteAfterDays": 7
        }
      },
      {
        "RuleName": "RDS-Weekly-30Days",
        "TargetBackupVaultName": "Standalone-RDS-Vault",
        "ScheduleExpression": "cron(0 1 ? * SUN *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 180,
        "Lifecycle": {
          "DeleteAfterDays": 30
        }
      },
      {
        "RuleName": "RDS-Monthly-365Days",
        "TargetBackupVaultName": "Standalone-RDS-Vault",
        "ScheduleExpression": "cron(0 2 1 * ? *)",
        "StartWindowMinutes": 60,
        "CompletionWindowMinutes": 180,
        "Lifecycle": {
          "DeleteAfterDays": 365
        }
      }
    ]
  }
}
EOF
```

Review before creating:

```bash
cat backup-plan.json
```

---

# 11. Create Backup Plan

Run:

```bash
aws backup create-backup-plan \
  --backup-plan file://backup-plan.json
```

The response will contain:

```json
{
    "BackupPlanId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
    "BackupPlanArn": "arn:aws:backup:...",
    "CreationDate": "..."
}
```

Save the returned plan ID:

```bash
export BACKUP_PLAN_ID="PASTE-BACKUP-PLAN-ID-HERE"
```

Verify:

```bash
echo "$BACKUP_PLAN_ID"
```

---

# 12. Verify Backup Plan

Run:

```bash
aws backup get-backup-plan \
  --backup-plan-id "$BACKUP_PLAN_ID"
```

For a cleaner output:

```bash
aws backup get-backup-plan \
  --backup-plan-id "$BACKUP_PLAN_ID" \
  --query 'BackupPlan.Rules[*].[RuleName,ScheduleExpression,Lifecycle.DeleteAfterDays]' \
  --output table
```

Expected:

```text
------------------------------------------------------------
|                     Backup Rules                         |
+-----------------------+-------------------------+--------+
| RDS-3xDaily-7Days     | cron(0 0,8,16 * * ? *) | 7      |
| RDS-Weekly-30Days     | cron(0 1 ? * SUN *)    | 30     |
| RDS-Monthly-365Days   | cron(0 2 1 * ? *)      | 365    |
+-----------------------+-------------------------+--------+
```

---

# 13. Create Backup Selection

The selection will automatically select RDS resources having:

```text
Backup=true
```

Create the selection file:

```bash
cat > backup-selection.json <<EOF
{
  "SelectionName": "Standalone-RDS-Selection",
  "IamRoleArn": "$BACKUP_ROLE_ARN",
  "ListOfTags": [
    {
      "ConditionType": "STRINGEQUALS",
      "ConditionKey": "Backup",
      "ConditionValue": "true"
    }
  ]
}
EOF
```

Review:

```bash
cat backup-selection.json
```

Create the selection:

```bash
aws backup create-backup-selection \
  --backup-plan-id "$BACKUP_PLAN_ID" \
  --backup-selection file://backup-selection.json
```

---

# 14. Verify Backup Selection

List selections:

```bash
aws backup list-backup-selections \
  --backup-plan-id "$BACKUP_PLAN_ID" \
  --output table
```

Get the selection ID:

```bash
export SELECTION_ID=$(aws backup list-backup-selections \
  --backup-plan-id "$BACKUP_PLAN_ID" \
  --query 'BackupSelectionsList[0].SelectionId' \
  --output text)
```

Verify:

```bash
echo "$SELECTION_ID"
```

Then:

```bash
aws backup get-backup-selection \
  --backup-plan-id "$BACKUP_PLAN_ID" \
  --selection-id "$SELECTION_ID"
```

Confirm:

```text
ConditionKey   = Backup
ConditionValue = true
```

---

# 15. Verify RDS Is Selected

Check protected resources:

```bash
aws backup list-protected-resources \
  --output table
```

Initially, the RDS may **not appear immediately** because the first scheduled backup has not run yet.

After a backup job successfully completes, check again.

---

# 16. Run an On-Demand Test Backup

Do this to verify the complete configuration without waiting for the schedule.

Start backup:

```bash
aws backup start-backup-job \
  --backup-vault-name "$BACKUP_VAULT" \
  --resource-arn "$RDS_ARN" \
  --iam-role-arn "$BACKUP_ROLE_ARN"
```

The response will contain:

```json
{
    "BackupJobId": "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

Save it:

```bash
export BACKUP_JOB_ID="PASTE-BACKUP-JOB-ID-HERE"
```

---

# 17. Monitor Backup Job

Check:

```bash
aws backup describe-backup-job \
  --backup-job-id "$BACKUP_JOB_ID" \
  --query '[State,PercentDone,ResourceType,ResourceArn]' \
  --output table
```

Possible states:

```text
CREATED
PENDING
RUNNING
COMPLETED
FAILED
ABORTED
```

The final expected state is:

```text
COMPLETED
```

---

# 18. Check Recovery Points

After the backup completes:

```bash
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name "$BACKUP_VAULT" \
  --output table
```

For a cleaner output:

```bash
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name "$BACKUP_VAULT" \
  --query 'RecoveryPoints[*].[RecoveryPointArn,ResourceType,CreationDate,Status,Lifecycle.DeleteAt]' \
  --output table
```

Confirm:

```text
Status = COMPLETED
```

---

# 19. Final Verification Checklist

Run:

### AWS identity

```bash
aws sts get-caller-identity
```

### RDS

```bash
aws rds describe-db-instances \
  --db-instance-identifier "$RDS_INSTANCE_ID" \
  --query 'DBInstances[0].[DBInstanceIdentifier,Engine,DBInstanceStatus,DBInstanceArn]' \
  --output table
```

### RDS tag

```bash
aws rds list-tags-for-resource \
  --resource-name "$RDS_ARN" \
  --output table
```

Confirm:

```text
Backup=true
```

### Backup vault

```bash
aws backup describe-backup-vault \
  --backup-vault-name "$BACKUP_VAULT"
```

### Backup plan

```bash
aws backup get-backup-plan \
  --backup-plan-id "$BACKUP_PLAN_ID"
```

### Backup selection

```bash
aws backup get-backup-selection \
  --backup-plan-id "$BACKUP_PLAN_ID" \
  --selection-id "$SELECTION_ID"
```

### Protected resource

```bash
aws backup list-protected-resources \
  --output table
```

### Recovery points

```bash
aws backup list-recovery-points-by-backup-vault \
  --backup-vault-name "$BACKUP_VAULT" \
  --output table
```

---

# 20. Final Backup Policy

The resulting configuration is:

```text
                    Standalone RDS
                          |
                     Backup=true
                          |
                     AWS Backup
                          |
              Standalone-RDS-Vault
                          |
        +-----------------+------------------+
        |                 |                  |
        |                 |                  |
      DAILY             WEEKLY             MONTHLY
        |                 |                  |
    3 snapshots        1 snapshot         1 snapshot
      per day            per week           per month
        |                 |                  |
      7 days             30 days           365 days
        |                 |                  |
    00:00 UTC           Sunday 01:00       1st 02:00
    08:00 UTC              UTC               UTC
    16:00 UTC
```

## Important Notes

### "Hourly Snapshot: 3 snapshots per day"

Although the requirement says **Hourly Snapshot**, 3 snapshots per day is technically **every 8 hours**, not hourly.

Therefore this configuration uses:

```text
00:00 UTC
08:00 UTC
16:00 UTC
```

If the customer's actual requirement is **every hour**, the schedule needs to be changed to:

```text
cron(0 * * * ? *)
```

which would create approximately **24 backups per day**.

### "1 month retention"

This configuration uses:

```text
30 days
```

for weekly backups.

### "1 year retention"

This configuration uses:

```text
365 days
```

for monthly backups.

### Production recommendation

Before enabling this in production, confirm:

1. Correct AWS Region
2. Correct RDS instance
3. Backup window
4. UTC vs IST schedule
5. 30-day interpretation of "1 month"
6. 365-day interpretation of "1 year"
7. IAM permissions of the CloudShell user/role
8. Whether encryption/KMS requirements exist
9. Whether cross-region backup is required
10. Whether AWS Backup Vault Lock is required

> **Do not delete or modify the existing backup configuration until the new backup plan has been tested successfully and the customer has confirmed the recovery point is usable.**

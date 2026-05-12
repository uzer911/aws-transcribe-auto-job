# AWS Transcribe Setup Guide

This guide explains how to set up the auto transcription project.
You can do it **manually** step by step OR **automatically** using CloudFormation.

---

# OPTION A — Manual Setup

## Step 1 — Create S3 Bucket

1. Go to **AWS Console → S3 → Create bucket**
2. Bucket name: `your-bucket-name`
3. Region: `us-east-1` (or your preferred region)
4. Click **Create bucket**
5. Open the bucket and create these 3 folders:
   - `input/` — upload mono audio here
   - `output/` — results will be saved here automatically
   - `analytics/` — upload stereo/2-channel audio here

---

## Step 2 — Create Transcribe Service Role

This role allows Amazon Transcribe to read/write your S3 bucket.

1. Go to **IAM → Roles → Create role**
2. Trusted entity: **AWS Service → Transcribe**
3. Attach policy: `AmazonTranscribeFullAccess`
4. Click **Next → Next → give it a name** e.g. `AmazonTranscribeServiceRole-my-role`
5. Click **Create role**
6. Open the role → **Add permissions → Create inline policy → JSON tab**
7. Paste this (replace `your-bucket-name`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    }
  ]
}
```

8. Click **Next → name it** `s3-access` → **Create policy**

---

## Step 3 — Create Lambda Execution Role

This role allows Lambda to call Transcribe and access S3.

1. Go to **IAM → Roles → Create role**
2. Trusted entity: **AWS Service → Lambda**
3. Attach policy: `AWSLambdaBasicExecutionRole`
4. Click **Next → Next → name it** `transcribe-lambda-role` → **Create role**
5. Open the role → **Add permissions → Create inline policy → JSON tab**
6. Paste this (replace `YOUR_ACCOUNT_ID` and `YOUR_TRANSCRIBE_ROLE_NAME`):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:ListBucket"],
      "Resource": [
        "arn:aws:s3:::your-bucket-name",
        "arn:aws:s3:::your-bucket-name/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "transcribe:StartTranscriptionJob",
        "transcribe:StartCallAnalyticsJob",
        "transcribe:GetCallAnalyticsJob",
        "transcribe:ListCallAnalyticsJobs"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": "arn:aws:iam::YOUR_ACCOUNT_ID:role/service-role/YOUR_TRANSCRIBE_ROLE_NAME"
    }
  ]
}
```

7. Click **Next → name it** `transcribe-access` → **Create policy**

---

## Step 4 — Create Standard Transcription Lambda

1. Go to **Lambda → Create function**
2. Name: `transcribe-audio`
3. Runtime: `Python 3.12`
4. Execution role: select `transcribe-lambda-role`
5. Click **Create function**
6. In the code editor, paste the code from `TranscribeFunction` file
7. Click **Deploy**
8. Go to **Configuration → General configuration → Edit**
   - Timeout: `3 minutes`
   - Click **Save**
9. Go to **Configuration → Environment variables → Edit**
   - Add: `INPUT_PREFIX` = `input/`
   - Add: `OUTPUT_PREFIX` = `output/results/`
   - Click **Save**

---

## Step 5 — Create Call Analytics Lambda

1. Go to **Lambda → Create function**
2. Name: `analytics-transcribe`
3. Runtime: `Python 3.12`
4. Execution role: select `transcribe-lambda-role`
5. Click **Create function**
6. In the code editor, paste the code from `TranscribeAnalyticsFunction` file
7. Click **Deploy**
8. Go to **Configuration → General configuration → Edit**
   - Timeout: `3 minutes`
   - Click **Save**
9. Go to **Configuration → Environment variables → Edit**
   - Add: `INPUT_PREFIX` = `analytics/`
   - Add: `OUTPUT_PREFIX` = `output/results/analytics/`
   - Add: `DATA_ACCESS_ROLE_ARN` = `arn:aws:iam::YOUR_ACCOUNT_ID:role/service-role/YOUR_TRANSCRIBE_ROLE_NAME`
   - Click **Save**

---

## Step 6 — Add S3 Triggers

**For transcribe-audio Lambda:**
1. Go to Lambda → `transcribe-audio` → **Add trigger**
2. Select **S3**
3. Bucket: `your-bucket-name`
4. Event type: **All object create events**
5. Prefix: `input/`
6. Suffix: `.mp3`
7. Click **Add**

**For analytics-transcribe Lambda:**
1. Go to Lambda → `analytics-transcribe` → **Add trigger**
2. Select **S3**
3. Bucket: `your-bucket-name`
4. Event type: **All object create events**
5. Prefix: `analytics/`
6. Suffix: `.mp3`
7. Click **Add**

---

## Step 7 — Test

- Upload a mono `.mp3` to `input/` → check **Transcribe → Transcription jobs**
- Upload a stereo `.mp3` to `analytics/` → check **Transcribe → Call Analytics**
- Results saved to `output/results/` and `output/results/analytics/`

---

## Step 8 — Cleanup When Done

- Delete both Lambda functions
- Delete IAM roles (`transcribe-lambda-role`, Transcribe service role)
- Empty and delete the S3 bucket

---
---

# OPTION B — Automatic Setup (CloudFormation)

Deploy everything in one command using `transcribe-two-trigger-stack.yaml`.

## What You Need to Change First

Open `transcribe-two-trigger-stack.yaml` and update these 3 default values:

| Parameter | Where | Change To |
|---|---|---|
| `ExistingBucketName` | Line 10 | Your S3 bucket name |
| `LambdaExecutionRoleName` | Line 14 | Any name you like |
| `TranscribeDataAccessRoleName` | Line 32 | Your Transcribe service role name |

> **How to find your Transcribe service role name?**
> IAM → Roles → search `AmazonTranscribeServiceRole` → copy the name

---

## Deploy Command

```bash
aws cloudformation deploy \
  --template-file transcribe-two-trigger-stack.yaml \
  --stack-name transcribe-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

---

## What CloudFormation Creates Automatically

- S3 folders (`input/`, `output/`, `analytics/`)
- IAM role with all required permissions
- Both Lambda functions with correct code
- S3 event notifications on both folders
- All wired together — no manual steps needed

---

## Cleanup CloudFormation Stack

```bash
aws cloudformation delete-stack --stack-name transcribe-stack --region us-east-1
```

---

# Quick Reference

| Upload to | Job Type | Results Location |
|---|---|---|
| `input/` | Transcription Job | `output/results/` |
| `analytics/` | Call Analytics Job | `output/results/analytics/` |

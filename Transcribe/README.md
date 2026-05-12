# AWS Transcribe Auto-Job Project

## What This Does
Automatically creates transcription jobs when you upload audio files to S3.

- Upload to `input/` → creates a **Transcription Job** (speech to text)
- Upload to `analytics/` → creates a **Call Analytics Job** (sentiment, agent vs customer, talk time)

---

## Before You Deploy — Change These 3 Things

### 1. Open `transcribe-two-trigger-stack.yaml`

Find the Parameters section at the top and update:

```yaml
ExistingBucketName:
  Default: transcript-911        # ← Change to YOUR bucket name

LambdaExecutionRoleName:
  Default: transcribe-lambda-role  # ← Change to any name you like

TranscribeDataAccessRoleName:
  Default: AmazonTranscribeServiceRole-transcribe-role-1  # ← Change to YOUR Transcribe service role name
```

> **How to find your Transcribe service role?**
> Go to AWS Console → IAM → Roles → search "AmazonTranscribeServiceRole"
> Copy the exact name (not the full ARN, just the name)

---

### 2. Open `TranscribeAnalyticsFunction`

Find line 15 and update the role ARN:

```python
DATA_ACCESS_ROLE = "arn:aws:iam::YOUR_ACCOUNT_ID:role/service-role/YOUR_TRANSCRIBE_ROLE_NAME"
```

Replace:
- `YOUR_ACCOUNT_ID` → your 12-digit AWS account ID
- `YOUR_TRANSCRIBE_ROLE_NAME` → same role name from step 1

> **How to find your Account ID?**
> Top right corner of AWS Console → click your name → Account ID is shown there

---

### 3. Open `Create inline policy`

Update the role ARN in the Resource field:

```json
"Resource": "arn:aws:iam::YOUR_ACCOUNT_ID:role/service-role/YOUR_TRANSCRIBE_ROLE_NAME"
```

---

## Deploy the Stack

Run this command in your terminal:

```bash
aws cloudformation deploy \
  --template-file transcribe-two-trigger-stack.yaml \
  --stack-name transcribe-stack \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

> Change `us-east-1` to your region if different.

---

## Test It

1. Go to S3 → your bucket → `input/` folder
2. Upload any `.mp3`, `.mp4`, or `.wav` file
3. Go to AWS Transcribe → Transcription jobs → you should see a new job

For analytics:
1. Upload `.mp3` to `analytics/` folder
2. Go to AWS Transcribe → Call Analytics → you should see a new job

---

## Folder Structure in S3

```
your-bucket/
├── input/              ← upload audio here for transcription
├── analytics/          ← upload audio here for call analytics
└── output/
    └── results/
        ├── job_*.json          ← transcription results
        └── analytics/
            └── analytics_*.json  ← call analytics results
```

---

## Files in This Project

| File | What it is |
|------|-----------|
| `transcribe-two-trigger-stack.yaml` | CloudFormation template — deploys everything |
| `TranscribeAnalyticsFunction` | Lambda code for call analytics jobs |
| `TranscribeFunction` | Lambda code for standard transcription jobs |
| `Create inline policy` | IAM policy to manually attach if needed |
| `Test_Sniffet` | JSON to test Lambda manually in AWS console |

---

## Common Errors & Fixes

| Error | Fix |
|-------|-----|
| `AccessDeniedException: StartCallAnalyticsJob` | IAM policy Resource is wrong — set it to `"*"` |
| `BadRequestException: S3 bucket can't be accessed` | Transcribe service role missing `s3:PutObject` |
| Lambda triggers but no job created | Check CloudWatch logs for the analytics Lambda |
| Upload fails in S3 console | Try uploading via AWS CLI instead |

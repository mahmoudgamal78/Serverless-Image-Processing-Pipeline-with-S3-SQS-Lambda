# 🖼️ Serverless Image Processing Pipeline

![AWS](https://img.shields.io/badge/AWS-Serverless-orange?logo=amazon-aws)
![Terraform](https://img.shields.io/badge/IaC-Terraform-7B42BC?logo=terraform)
![Python](https://img.shields.io/badge/Python-3.11-blue?logo=python)
![License](https://img.shields.io/badge/License-MIT-green)

A fully serverless, event-driven image processing pipeline on AWS. Upload an image → get back a thumbnail, a watermarked full-resolution version, and rich metadata — all without managing a single server.

---

## 📐 Architecture

```
Client
  │
  ▼
API Gateway  ──►  Lambda (presign)  ──►  S3 Pre-signed URL
                                               │
                                        Client uploads image
                                               │
                                               ▼
                                    S3 Source Bucket
                                    (s3:ObjectCreated)
                                               │
                                               ▼
                                         SQS Queue
                                        (+ DLQ for failures)
                                               │
                                               ▼
                                    Lambda (validate)
                                               │
                                               ▼
                                   Step Functions Workflow
                          ┌────────────────────────────────────┐
                          │  Validate                          │
                          │      ↓                             │
                          │  Parallel Processing               │
                          │  ├── Resize → Thumbnail (200×200)  │
                          │  └── Resize → Full-res             │
                          │           └── Watermark            │
                          │      ↓                             │
                          │  Store Metadata (DynamoDB)         │
                          │      ↓                             │
                          │  Notify (SNS)                      │
                          └────────────────────────────────────┘
                                               │
                                               ▼
                                   S3 Destination Bucket
                                               │
                                               ▼
                                   CloudFront CDN (Global)
```

---

## ☁️ AWS Services Used

| Service | Role |
|---|---|
| **S3** | Source bucket (uploads) + Destination bucket (processed images) |
| **SQS + DLQ** | Decouples S3 events from processing; retries + dead-letter queue |
| **Lambda** | Presign, Validate, Resize, Watermark, Store, Notify (6 functions) |
| **Lambda Layers** | Pillow library shared across functions |
| **Step Functions** | Orchestrates validate → resize → watermark → store → notify |
| **API Gateway** | HTTP API for generating pre-signed upload URLs |
| **DynamoDB** | Image metadata: dimensions, status, output keys, timestamps |
| **CloudFront** | Global CDN for serving processed images (OAC, no public S3) |
| **SNS** | Email notifications on job success or failure |

---

## 🗂️ Project Structure

```
image-pipeline/
├── main.tf                        # Root Terraform module
├── variables.tf                   # All input variables
├── outputs.tf                     # Stack outputs
├── terraform.tfvars.example       # Example config
│
├── modules/
│   ├── s3/                        # Buckets, lifecycle rules, event notifications
│   ├── sqs/                       # Main queue + DLQ
│   ├── dynamodb/                  # Metadata table with GSI on status
│   ├── sns/                       # Notification topic + email subscription
│   ├── lambda/                    # 6 functions + IAM role + Pillow layer
│   ├── stepfunctions/             # State machine definition + IAM
│   ├── apigateway/                # HTTP API v2 for pre-signed URLs
│   └── cloudfront/                # CDN distribution + OAC policy
│
├── lambda_functions/
│   ├── presign/handler.py         # Generates S3 pre-signed PUT URL
│   ├── validate/handler.py        # Validates image type/size, starts workflow
│   ├── resize/handler.py          # Pillow: thumbnail + full-res resize
│   ├── watermark/handler.py       # Pillow: semi-transparent text watermark
│   ├── store/handler.py           # Writes final metadata to DynamoDB
│   └── notify/handler.py          # Publishes SNS success/failure notification
│
└── step_functions/
    └── workflow.json.tpl          # State machine ASL (Terraform template)
```

---

## 🚀 Deployment

### Prerequisites

- Terraform >= 1.5.0
- AWS CLI configured (`aws configure`)
- Python 3.11+

### 1. Build the Pillow Lambda Layer

```bash
mkdir -p layers/python
pip install pillow \
  -t layers/python/ \
  --platform manylinux2014_x86_64 \
  --implementation cp \
  --python-version 3.11 \
  --only-binary=:all:

cd layers && zip -r pillow_layer.zip python/ && cd ..
```

### 2. Configure Variables

```bash
cp terraform.tfvars.example terraform.tfvars
# Edit terraform.tfvars — set your email for SNS notifications
```

### 3. Deploy

```bash
terraform init
terraform plan
terraform apply
```

### 4. Confirm SNS Email

Check your inbox and confirm the subscription email from AWS.

---

## 📤 Usage

```bash
# 1. Get a pre-signed upload URL
RESPONSE=$(curl -s -X POST \
  "$(terraform output -raw presigned_url_api_endpoint)/upload/presign" \
  -H "Content-Type: application/json" \
  -d '{"filename":"photo.jpg","content_type":"image/jpeg"}')

UPLOAD_URL=$(echo $RESPONSE | jq -r '.upload_url')
IMAGE_ID=$(echo $RESPONSE | jq -r '.image_id')

# 2. Upload directly to S3
curl -X PUT "$UPLOAD_URL" \
  -H "Content-Type: image/jpeg" \
  --upload-file photo.jpg

# 3. Access processed results via CloudFront
CF=$(terraform output -raw cloudfront_domain)
echo "Thumbnail : https://$CF/processed/thumbnails/$IMAGE_ID/photo_thumb.jpg"
echo "Full-res  : https://$CF/processed/fullres/$IMAGE_ID/photo_fullres.jpg"
```

---

## 🔑 Key Design Decisions

| Decision | Reason |
|---|---|
| S3 → SQS (not S3 → Lambda directly) | SQS enables retries, DLQ, back-pressure, and decoupling |
| Lambda Layer for Pillow | Keeps deployment packages small; shared across 3 functions |
| Step Functions STANDARD | Full audit history, per-step retries, easy failure handling |
| Parallel resize branches | Thumbnail and full-res processed concurrently |
| CloudFront + OAC | S3 stays private; CloudFront authenticates with SigV4 |
| DynamoDB PAY_PER_REQUEST | No capacity planning for variable image upload rates |
| SQS long polling (20s) | Reduces empty receive calls and cost |

---

## 📊 DynamoDB Schema

| Attribute | Type | Description |
|---|---|---|
| `image_id` | String (PK) | UUID generated at upload time |
| `created_at` | String (SK) | ISO 8601 timestamp |
| `status` | String | `PENDING` → `PROCESSING` → `COMPLETED` / `FAILED` |
| `source_key` | String | S3 key in source bucket |
| `thumbnail_key` | String | S3 key in destination bucket |
| `fullres_key` | String | S3 key in destination bucket |
| `content_type` | String | Image MIME type |
| `file_size` | Number | File size in bytes |
| `completed_at` | String | Processing completion timestamp |

**GSI:** `status-index` — query all images by status.

---

## 🧹 Cleanup

```bash
terraform destroy
```

> `force_destroy = true` is set on S3 buckets for easy dev teardown. Remove for production.

---

## 📚 Learning Outcomes

- ✅ Event-driven architecture with S3 notifications → SQS
- ✅ SQS decoupling for resilience and retries
- ✅ Lambda Layers for large dependencies
- ✅ Step Functions for multi-step serverless workflows
- ✅ S3 lifecycle policies (Standard → IA → Glacier → Expire)
- ✅ CloudFront with Origin Access Control (OAC)

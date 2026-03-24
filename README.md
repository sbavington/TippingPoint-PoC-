# TippingPoint PoC — Discovery & Scoping App

A full-stack AWS serverless application for managing TippingPoint IPS Proof-of-Concept engagements. SEs and AMs use this tool to capture discovery data, generate PDF build sheets, and track PoC status across the team.

---

## Architecture

```
CloudFront → S3 (index.html, login.html)
                ↓
         API Gateway (REST)
                ↓
           Lambda (Python 3.12)
                ↓
     ┌──────────┬──────────┐
  DynamoDB   DynamoDB   DynamoDB      S3
 (documents) (users)  (sessions)  (PDFs)
```

| Resource | Purpose |
|---|---|
| **S3 (frontend)** | Hosts `index.html` and `login.html` as a static website |
| **CloudFront** | CDN / HTTPS termination in front of the S3 website |
| **API Gateway** | REST API, regional endpoint, `prod` stage |
| **Lambda** | Single-function handler for all routes |
| **DynamoDB – documents** | Stores PoC form data and metadata |
| **DynamoDB – users** | Stores user accounts with hashed passwords |
| **DynamoDB – sessions** | Token-based sessions with TTL auto-expiry (7 days) |
| **S3 (documents)** | Private bucket for generated PDF build sheets |

---

## Project Structure

```
tippingpoint-poc/
├── cloudformation/
│   └── tippingpoint-poc-corrected.yaml   # Main IaC template
├── frontend/
│   ├── index.html                        # Main app (library + form + admin)
│   └── login.html                        # Login / change-password page
└── README.md
```

---

## Prerequisites

- AWS CLI configured (`aws configure`) with sufficient IAM permissions
- An S3 bucket to store the CloudFormation template (or deploy inline)
- Git

---

## Deploy

### 1. Clone the repository

```bash
git clone https://github.com/YOUR_ORG/tippingpoint-poc.git
cd tippingpoint-poc
```

### 2. Deploy the CloudFormation stack

```bash
aws cloudformation deploy \
  --template-file cloudformation/tippingpoint-poc-corrected.yaml \
  --stack-name tippingpoint-poc \
  --capabilities CAPABILITY_IAM \
  --region us-east-2
```

### 3. Get the stack outputs

```bash
aws cloudformation describe-stacks \
  --stack-name tippingpoint-poc \
  --query "Stacks[0].Outputs" \
  --output table \
  --region us-east-2
```

Note the following values — you'll need them next:

| Output Key | Used for |
|---|---|
| `FrontendBucketName` | Uploading frontend files |
| `CloudFrontURL` | App URL to share with users |
| `CloudFrontDistributionId` | Cache invalidation after updates |
| `ApiEndpoint` | Hardcoded into `index.html` and `login.html` |

### 4. Set the API endpoint in the frontend files

Replace the `API_URL` constant in both `frontend/index.html` and `frontend/login.html`:

```js
const API_URL = 'https://<RestApiId>.execute-api.us-east-2.amazonaws.com/prod';
```

### 5. Upload frontend files to S3

```bash
BUCKET=$(aws cloudformation describe-stacks \
  --stack-name tippingpoint-poc \
  --query "Stacks[0].Outputs[?OutputKey=='FrontendBucketName'].OutputValue" \
  --output text --region us-east-2)

aws s3 cp frontend/index.html s3://$BUCKET/index.html --content-type "text/html"
aws s3 cp frontend/login.html s3://$BUCKET/login.html --content-type "text/html"
```

### 6. Invalidate CloudFront cache

```bash
DIST_ID=$(aws cloudformation describe-stacks \
  --stack-name tippingpoint-poc \
  --query "Stacks[0].Outputs[?OutputKey=='CloudFrontDistributionId'].OutputValue" \
  --output text --region us-east-2)

aws cloudfront create-invalidation \
  --distribution-id $DIST_ID \
  --paths "/*"
```

### 7. Create the first admin user

Use the AWS Console or CLI to put an item directly into the Users DynamoDB table, or write a one-off script:

```bash
USERS_TABLE=$(aws cloudformation describe-stacks \
  --stack-name tippingpoint-poc \
  --query "Stacks[0].Outputs[?OutputKey=='UsersTableName'].OutputValue" \
  --output text --region us-east-2)

# Python helper — run once to seed the admin account
python3 - <<'EOF'
import boto3, hashlib, uuid, secrets
from datetime import datetime, timezone

table = boto3.resource('dynamodb', region_name='us-east-2').Table('REPLACE_WITH_TABLE_NAME')
pw = 'ChangeMe123!'   # user will be forced to change on first login
table.put_item(Item={
    'user_id': str(uuid.uuid4()),
    'email': 'admin@yourcompany.com',
    'name': 'Admin',
    'role': 'admin',
    'status': 'active',
    'must_change': True,
    'password_hash': hashlib.sha256(pw.encode()).hexdigest(),
    'created_at': datetime.now(timezone.utc).isoformat()
})
print(f'Admin created. Temp password: {pw}')
EOF
```

---

## Updating the Stack

### Frontend-only change (HTML/CSS/JS)

```bash
# 1. Edit frontend/index.html or frontend/login.html
# 2. Upload
aws s3 cp frontend/index.html s3://$BUCKET/index.html --content-type "text/html"
aws s3 cp frontend/login.html s3://$BUCKET/login.html --content-type "text/html"
# 3. Invalidate cache
aws cloudfront create-invalidation --distribution-id $DIST_ID --paths "/*"
```

### Backend / infrastructure change

```bash
aws cloudformation deploy \
  --template-file cloudformation/tippingpoint-poc-corrected.yaml \
  --stack-name tippingpoint-poc \
  --capabilities CAPABILITY_IAM \
  --region us-east-2
```

---

## Tear Down

```bash
# Empty the S3 buckets first (CloudFormation cannot delete non-empty buckets)
aws s3 rm s3://$BUCKET --recursive

DOCS_BUCKET=$(aws cloudformation describe-stacks \
  --stack-name tippingpoint-poc \
  --query "Stacks[0].Outputs[?OutputKey=='DocumentsBucketName'].OutputValue" \
  --output text --region us-east-2)
aws s3 rm s3://$DOCS_BUCKET --recursive

# Delete the stack
aws cloudformation delete-stack --stack-name tippingpoint-poc --region us-east-2
```

---

## API Reference

All endpoints require an `Authorization: Bearer <token>` header except `/auth/login`.

| Method | Path | Auth | Description |
|---|---|---|---|
| `POST` | `/auth/login` | ❌ | Login, returns token |
| `GET` | `/auth/me` | ✅ | Get current user |
| `POST` | `/auth/logout` | ✅ | Invalidate session token |
| `POST` | `/auth/change-password` | ✅ | Change password |
| `GET` | `/docs` | ✅ | List all PoC documents |
| `POST` | `/docs` | ✅ | Create new PoC |
| `PUT` | `/docs/{id}` | ✅ | Update existing PoC |
| `DELETE` | `/docs/{id}` | ✅ | Delete PoC and its PDF |
| `GET` | `/docs/{id}/download` | ✅ | Get presigned S3 URL for PDF |
| `GET` | `/users` | ✅ Admin | List all users |
| `POST` | `/users` | ✅ Admin | Create user |
| `PUT` | `/users/{id}` | ✅ Admin | Update user (role/status/name/password) |
| `DELETE` | `/users/{id}` | ✅ Admin | Delete user |

---

## Notes

- Passwords are hashed with SHA-256. For production, upgrade to bcrypt or Argon2.
- Sessions expire after 7 days via DynamoDB TTL.
- CloudFront has `DefaultTTL=0` for development convenience — increase for production.
- The PDF bucket is private; downloads are served via 5-minute presigned URLs.

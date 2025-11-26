# CloudTrail → SNS → SQS → Fluentd → Node.js → SQLite Pipeline

This project demonstrates an end‑to‑end log ingestion pipeline:

AWS CloudTrail  
→ S3  
→ SNS
→ SQS  
→ Fluentd (running in Docker)  
→ Node.js app (REST API)  
→ SQLite database

The goal is to collect CloudTrail/S3 event notifications and store them in a local SQLite DB for testing, analytics, or building downstream processors.

---

## 🚀 Architecture Overview

1. **CloudTrail** writes logs to an S3 bucket.  
2. **S3 Event Notification** sends events to the **SQS queue**.  
3. **Fluentd** reads messages from SQS using `fluent-plugin-sqs`.  
4. Fluentd parses SNS/S3/CloudTrail JSON and POSTs the full log to:  
   ```
   POST /ingest
   ```
   on the Node.js application.
5. The Node.js service writes events into a SQLite database  
   (`cloudtrail.db` inside `app/data/`).

---

## 📁 Project Structure

```
cloudtrail-project/
├── docker-compose.yml
├── log-generator.sh         # optional load generator
├── app/
│   ├── server.js
│   ├── package.json
│   └── Dockerfile
└── fluentd/
    ├── fluent.conf
    └── Dockerfile
```

---

## 🧩 Components

### 1. Node.js App
Runs an Express server that exposes `/ingest` and writes logs into SQLite:

- Creates DB file at `/usr/src/app/data/cloudtrail.db`
- Inserts every incoming event
- Logs requests to console for easy debugging

### 2. Fluentd
Reads messages from your AWS SQS queue using:

```
fluent-plugin-sqs
fluent-plugin-out-http
```

It parses:

- SQS body → SNS  
- SNS Message → CloudTrail/S3 event JSON  

Then forwards to the Node app.

---

## 🐳 Running with Docker Compose

### 1. Export AWS credentials:

```
export AWS_ACCESS_KEY_ID="YOUR_KEY"
export AWS_SECRET_ACCESS_KEY="YOUR_SECRET"
```

### 2. Start the stack:

```
docker-compose up -d --build
```

### 3. Check logs:

```
docker logs app --tail 20
docker logs fluentd --tail 20
```

You should see logs entering the DB.

---

## 🔍 Testing the Pipeline

Upload a file to your CloudTrail S3 bucket:

```
echo '{"msg":"pipeline test"}' > test.json
aws s3 cp test.json s3://YOUR-CLOUDTRAIL-BUCKET/
```

Wait 5–10 seconds, then check SQLite:

```
sqlite3 app/data/cloudtrail.db "SELECT COUNT(*) FROM logs;"
```

If everything works, the number increases with every S3 upload.

---

## ⚙️ Optional: Log Generator

A small script that uploads 1 file per second:

```
./log-generator.sh
```

---

## 🧹 Clean Up

Stop containers:

```
docker-compose down
```

Remove volumes (including SQLite DB):

```
docker-compose down -v
```

---

## 🙌 Notes

- You may replace SQLite with PostgreSQL if moving to production.  
- Docker Compose allows scaling the app service for parallel inserts.  
- Fluentd config can be extended for transformations, filtering, or routing.

---

## 💬 Need Improvements?

I can also generate:

- Architecture diagram  
- Terraform / CloudFormation  
- A TypeScript version  
- A production-ready version with PostgreSQL, retries, and metrics  

Just ask!


---

# 🏗️ A) AWS Manual Configuration Steps

Below are the exact steps to manually configure AWS so S3 ➝ SNS ➝ SQS ➝ Fluentd pipeline works end‑to‑end.

---

## **Step‑1: Create an S3 Bucket**
1️⃣ Go to **AWS Console → S3**  
2️⃣ Click **Create bucket**  
3️⃣ Example bucket name:

```
aws-cloudtrail-logs-<ACCOUNT-ID>-project
```

4️⃣ Region: **ap-south-1**  
5️⃣ Block Public Access: **Enabled**  
6️⃣ Click **Create bucket**

---

## **Step‑2: Enable CloudTrail & Send Logs to S3**
1️⃣ Go to **CloudTrail → Trails**  
2️⃣ Click **Create trail**

- **Name:** `cloudtrail-project-trail`  
- Enable **Management** & **Data** events  
- Select the S3 bucket created above  

Click **Create trail** 🚀

---

## **Step‑3: Create SNS Topic**
1️⃣ Go to **SNS → Topics → Create topic**  
2️⃣ Choose **Standard**  
3️⃣ Name:

```
cloudtrail-log-topic
```

Click **Create Topic** ✔️

---

## **Step‑4: Create SQS Queue**
1️⃣ Go to **SQS → Create Queue**  
2️⃣ Name:

```
1k-SQS-Notification
```

3️⃣ Type: **Standard Queue**  
4️⃣ Leave all defaults → **Create** ✔️

---

## **Step‑5: Subscribe SQS to SNS**
1️⃣ Go to **SNS** → open topic **cloudtrail-log-topic**  
2️⃣ Click **Create subscription**  
3️⃣ Set:

- **Protocol:** Amazon SQS  
- **Endpoint:** select `1k-SQS-Notification`

Click **Create Subscription**

---

## **Step‑6: Allow SNS to Send Messages to SQS**
1️⃣ Go to **SQS → 1k-SQS-Notification → Permissions → Access Policy → Edit**  
2️⃣ Add this:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "sns.amazonaws.com"},
    "Action": "sqs:SendMessage",
    "Resource": "arn:aws:sqs:ap-south-1:<ACCOUNT-ID>:1k-SQS-Notification",
    "Condition": {
      "ArnEquals": {
        "aws:SourceArn": "arn:aws:sns:ap-south-1:<ACCOUNT-ID>:cloudtrail-log-topic"
      }
    }
  }]
}
```

Replace `<ACCOUNT-ID>` with your AWS account ID.

---

## **Step‑7: Enable S3 Event Notifications**
1️⃣ Go to **S3 → Bucket → Properties**  
2️⃣ Scroll to **Event notifications → Create**  
3️⃣ Configure:

- **Name:** `SendS3ToSNS`  
- **Event type:** PUT  
- **Destination:** SNS  
- Select: **cloudtrail-log-topic**  

Click **Save** ✔️

---

### 🎯 Final Flow

```
S3 (PUT event)
   ↓
SNS Topic
   ↓
SQS Queue
   ↓
Fluentd Container
   ↓
Node.js API (/ingest)
   ↓
SQLite Database
```

The pipeline is now fully active!  

# 🚀 AWS Serverless ETL Pipeline
### *(Complete Event-Driven Data Pipeline using Amazon S3, AWS Lambda, AWS Glue & Amazon Athena)*

![AWS_Serverless_ETL_Pipeline](https://github.com/user-attachments/assets/686f4d13-46cd-4fb9-a841-c30056154300)

---

## 🧠 Project Overview

This project demonstrates how to build a **fully serverless, event-driven ETL (Extract, Transform, Load) pipeline** using **Amazon Web Services (AWS)**.

It automatically:
1. Ingests raw **JSON transaction data** into **Amazon S3**.
2. Uses **AWS Lambda** to **flatten and transform** JSON into **Parquet format**.
3. Stores transformed data in an **S3 Data Lake**.
4. Triggers an **AWS Glue Crawler** to update the **AWS Glue Data Catalog**.
5. Enables **SQL querying and analytics** using **Amazon Athena**.

The pipeline is **100% event-driven**, **scalable**, and **cost-efficient**, requiring no server management.

---

## 🧩 Architecture Overview

### 🔄 Workflow Steps

#### 🪣 Amazon S3 (Raw Data)
- Transactional data in **JSON** format is uploaded to an S3 bucket.
- This triggers an **AWS Lambda** function via **S3 event notification**.

#### 🧠 AWS Lambda (Transformation)
- Lambda reads the JSON file from S3.
- It **flattens** the nested JSON data structure using *pandas*.
- The transformed data is converted into **Parquet** for optimized querying.
- Parquet files are written to another S3 bucket (Data Lake).

#### 🪣 Amazon S3 (Data Lake)
- Stores the **transformed Parquet files**.
- Another S3 event trigger invokes the **AWS Glue Crawler** for schema updates.

#### 🧭 AWS Glue Crawler (Data Catalog)
- Automatically **crawls** the Parquet data stored in S3.
- Updates the **AWS Glue Data Catalog**, creating or refreshing tables and schemas dynamically.

#### 📊 Amazon Athena (Data Analysis)
- SQL queries can be executed **directly on S3** through Athena.
- Enables **serverless analytics** without any infrastructure management.

---

## 🪣 Amazon S3 Bucket Structure

**Bucket Name:** `navina-etl-project`

The S3 bucket serves as both the **data source** and **data lake** for this ETL pipeline.

```
navina-etl-project/
│
├── orders_json_inputdata/         # Raw JSON transactional data (Lambda trigger source)
│     ├── order_2025_11_03.json
│     └── order_2025_11_04.json
│
└── orders_parquet_datalakeetl/    # Transformed Parquet data (ETL output)
      ├── orders_ETL_20251103_094500.parquet
      └── orders_ETL_20251103_150830.parquet
```

### 🔄 Data Flow Summary

1️⃣ JSON file uploaded → Triggers Lambda function  
2️⃣ Lambda flattens & converts JSON → Parquet  
3️⃣ Parquet file stored in Data Lake (S3)  
4️⃣ AWS Glue Crawler updates schema in Data Catalog  
5️⃣ Amazon Athena queries transformed data  

---

## 🛠️ AWS Services Used

| Service | Purpose |
|----------|----------|
| **Amazon S3** | Data lake storage for raw JSON and transformed Parquet datasets. |
| **AWS Lambda** | Event-driven ETL — flattens JSON, converts to Parquet, triggers Glue crawler. |
| **AWS Glue Crawler** | Discovers schema and updates Data Catalog automatically. |
| **Amazon Athena** | Enables interactive SQL queries over Parquet data in S3. |
| **Amazon CloudWatch** | Monitors Lambda logs and crawler execution for debugging. |

---

## ⚙️ Workflow Summary

| Step | AWS Service | Description |
|------|--------------|--------------|
| 1️⃣ | **Amazon S3 (Raw)** | JSON uploaded → triggers Lambda. |
| 2️⃣ | **AWS Lambda** | Reads, flattens, converts JSON → Parquet. |
| 3️⃣ | **Amazon S3 (Data Lake)** | Stores transformed Parquet files. |
| 4️⃣ | **AWS Glue Crawler** | Updates metadata in Glue Catalog. |
| 5️⃣ | **Amazon Athena** | Queries Parquet data with SQL. |

---

## 🧑‍💻 Lambda Function Code

```python
import json
import boto3
import pandas as pd
import io
from datetime import datetime

def flatten(data):
    orders_data = []
    for order in data:
        for product in order['products']:
            row_orders = {
                "order_id": order["order_id"],
                "order_date": order["order_date"],
                "total_amount": order["total_amount"],
                "customer_id": order["customer"]["customer_id"],
                "customer_name": order["customer"]["name"],
                "email": order["customer"]["email"],
                "address": order["customer"]["address"],
                "product_id": product["product_id"],
                "product_name": product["name"],
                "category": product["category"],
                "price": product["price"],
                "quantity": product["quantity"]
            }
            orders_data.append(row_orders)
    return pd.DataFrame(orders_data)

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    response = s3.get_object(Bucket=bucket, Key=key)
    data = json.loads(response['Body'].read().decode('utf-8'))

    df = flatten(data)
    parquet_buffer = io.BytesIO()
    df.to_parquet(parquet_buffer, index=False, engine='pyarrow')

    timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
    parquet_key = f'orders_parquet_datalakeetl/orders_ETL_{timestamp}.parquet'
    s3.put_object(Bucket=bucket, Key=parquet_key, Body=parquet_buffer.getvalue())

    glue = boto3.client('glue')
    glue.start_crawler(Name='aws_etl_pipeline_crawler')

    return {'statusCode': 200, 'body': json.dumps('Lambda ETL executed successfully!')}
```

---

## 📦 Requirements

```
pandas==2.2.3
pyarrow==17.0.0
boto3==1.35.30
```

---

## 📂 Sample JSON Input

```json
[
  {
    "order_id": "ORD123",
    "order_date": "2025-11-02",
    "total_amount": 249.99,
    "customer": {
      "customer_id": "CUST001",
      "name": "John Doe",
      "email": "john@example.com",
      "address": "123 Main Street"
    },
    "products": [
      {
        "product_id": "PROD100",
        "name": "Wireless Mouse",
        "category": "Electronics",
        "price": 49.99,
        "quantity": 2
      },
      {
        "product_id": "PROD101",
        "name": "Keyboard",
        "category": "Electronics",
        "price": 150.00,
        "quantity": 1
      }
    ]
  }
]
```

---

## 🪜 Deployment Steps

1️⃣ **Create S3 Buckets**
- `orders_json_inputdata/` → Input JSON files.
- `orders_parquet_datalakeetl/` → Output Parquet files.

2️⃣ **Create IAM Role for Lambda**
Attach:
- `AmazonS3FullAccess`
- `AWSGlueServiceRole`
- `CloudWatchLogsFullAccess`

3️⃣ **Deploy Lambda Function**
- Runtime: *Python 3.12*
- Add code and dependencies.

4️⃣ **Configure S3 Trigger**
- Event: *ObjectCreated*
- Source: *Raw data bucket*
- Target: *Lambda Function*

5️⃣ **Create AWS Glue Crawler**
- Source: *Parquet S3 path*
- Target Database: *aws_etl_database*

6️⃣ **Query via Amazon Athena**
```sql
SELECT customer_name, SUM(total_amount) AS total_spent
FROM orders_parquet_datalakeetl
GROUP BY customer_name
ORDER BY total_spent DESC;
```

---

## 🌟 Key Highlights

✅ Fully Serverless Architecture  
✅ Event-Driven Automation  
✅ Parquet-Optimized Analytics  
✅ Low-Cost Pay-Per-Use Model  
✅ Easy to Scale and Extend  

---

## 💡 Future Enhancements

- Add **data validation layers**.  
- Orchestrate using **AWS Step Functions**.  
- Integrate **QuickSight** dashboards.  
- Add **SNS notifications** after ETL completion.  

---

## 👩‍💻 Author

**Navi** — *Investigation Associate @ Amazon*  
💼 Passionate about Data Analytics & exploring AWS Data Analytics Services

---

## 📸 AWS Console Snapshots
(Visual proof of S3, Lambda, Glue, and Athena in action)

<img width="1905" height="825" alt="1 Amazon S3" src="https://github.com/user-attachments/assets/1f77272d-3f32-4bf1-8971-9eade5f86968" />

<img width="1885" height="820" alt="2 AWS Lambda" src="https://github.com/user-attachments/assets/a302ce4f-371a-45ef-b049-f3b6ab6c8b4c" />

<img width="1886" height="820" alt="3 AWS Glue" src="https://github.com/user-attachments/assets/f8619bd4-2c04-4845-83a6-48c31b755d9b" />

<img width="1885" height="820" alt="4 Amazon Athena - Query" src="https://github.com/user-attachments/assets/bb3247c4-f07b-44ae-bf01-f29e775a26e2" />

<img width="1886" height="820" alt="4 Amazon Athena - Output" src="https://github.com/user-attachments/assets/0e9a5f9a-713c-4aff-8722-2ebca061f23d" />

<img width="1885" height="820" alt="CloudWatch" src="https://github.com/user-attachments/assets/8364924a-813f-428d-b115-4a85b06a19fa" />

<img width="1885" height="820" alt="IAM" src="https://github.com/user-attachments/assets/f3ddedc4-a967-4f6e-b98e-dc2789df6571" />







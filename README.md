# Live Replication Between S3 Bucket Folder and EOS Bucket Folder

This guide outlines the process of setting up live replication between an AWS S3 bucket folder and an EOS (E2E Object Storage) bucket folder using AWS Lambda.

## Prerequisites
- AWS account with access to S3 and Lambda
- EOS (E2E Object Storage) bucket
- IAM Role with necessary permissions

## 1. IAM Role Creation with S3 Full Access

### Create IAM Role
1. Navigate to AWS IAM Console.
2. Click on **Roles** → **Create Role**.
3. Choose **AWS Service** → **Lambda**.
4. Attach the following policy:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetObject"
            ],
            "Resource": [
                "arn:aws:s3:::your-bucket-name",
                "arn:aws:s3:::your-bucket-name/*"
            ]
        }
    ]
}
```
5. Name the role **LambdaS3FullAccess** and create it.
6. Attach this role to your Lambda function.

## 2. Create an AWS Lambda Function
1. Open AWS Lambda Console → **Create Function**.
2. Choose **Author from scratch**.
3. Set runtime to **Python 3.10**.
4. Select the IAM role created earlier.
5. Click **Create Function**.

## 3. Setup a Test Event
1. Open AWS Lambda Console → Select your function.
2. Click **Test** → **Create new test event**.
3. Use the following JSON:

```json
{
  "Records": [
    {
      "eventSource": "aws:s3",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": {
          "name": "S3_Bucket-name"
        },
        "object": {
          "key": "Folder-name/Object"
        }
      }
    }
  ]
}
```
4. Save the test event.

## 4. Lambda Deployment with Dependencies

### Prepare Lambda Package
```sh
mkdir -p lambda_package
cd lambda_package
pip install minio -t lambda_package/
cd lambda_package/
```

### Add Function Code
```sh
vim lambda_function.py
```
Paste the following code:
```python
import os
import io
import json
import boto3
from minio import Minio
from minio.error import S3Error


# AWS S3 Client
s3_client = boto3.client('s3')


# MinIO Credentials
MINIO_ENDPOINT = os.getenv("MINIO_ENDPOINT", "objectstore.e2enetworks.net")
MINIO_ACCESS_KEY = os.getenv("MINIO_ACCESS_KEY")
MINIO_SECRET_KEY = os.getenv("MINIO_SECRET_KEY")
MINIO_BUCKET = os.getenv("MINIO_BUCKET")
S3_BUCKET = os.getenv("S3_BUCKET")
S3_FOLDER = os.getenv("S3_FOLDER")
MINIO_FOLDER = os.getenv("MINIO_FOLDER")


# MinIO Client
minio_client = Minio(
   MINIO_ENDPOINT,
   access_key=MINIO_ACCESS_KEY,
   secret_key=MINIO_SECRET_KEY,
   secure=True
)


def list_minio_objects():
   """Get a list of objects only within the MinIO sync folder."""
   objects = minio_client.list_objects(MINIO_BUCKET, prefix=f"{MINIO_FOLDER}/", recursive=True)
   return {obj.object_name for obj in objects}


def sync_folders():
   """Sync objects from S3 to MinIO and remove deleted objects only within the folder."""
  
   # Fetch objects in S3
   s3_objects = s3_client.list_objects_v2(Bucket=S3_BUCKET, Prefix=S3_FOLDER)
   s3_keys = set()


   if 'Contents' in s3_objects:
       for obj in s3_objects['Contents']:
           relative_path = obj['Key'][len(S3_FOLDER):].lstrip('/')
           s3_keys.add(relative_path)
           minio_object_path = f"{MINIO_FOLDER}/{relative_path}"


           # Check if the object exists in MinIO
           try:
               minio_client.stat_object(MINIO_BUCKET, minio_object_path)
               print(f"Skipping {minio_object_path} (Already exists)")
               continue
           except S3Error:
               pass  # Proceed with upload if file is missing


           # Stream S3 object into memory
           response = s3_client.get_object(Bucket=S3_BUCKET, Key=obj['Key'])
           data = io.BytesIO(response['Body'].read())


           # Upload directly to MinIO
           minio_client.put_object(MINIO_BUCKET, minio_object_path, data, length=len(data.getvalue()))
           print(f"Uploaded {obj['Key']} to MinIO as {minio_object_path}")


   # Find objects in MinIO **only within the sync folder** that do not exist in S3 and delete them
   minio_objects = list_minio_objects()
   s3_keys_with_prefix = {f"{MINIO_FOLDER}/{key}" for key in s3_keys}
   objects_to_delete = minio_objects - s3_keys_with_prefix


   for obj in objects_to_delete:
       minio_client.remove_object(MINIO_BUCKET, obj)
       print(f"Deleted {obj} from MinIO (No longer in S3)")


def lambda_handler(event, context):
   try:
       sync_folders()
   except Exception as e:
       print(f"Error: {e}")
       return {'statusCode': 500, 'body': json.dumps('Replication Failed')}
  
   return {'statusCode': 200, 'body': json.dumps('Replication Successful')}

```

### Zip and Upload
```sh
zip -r ../lambda_function.zip .
cd ..
ls
```

Upload `lambda_function.zip` to AWS Lambda.

## 5. Set Environment Variables in AWS Lambda
1. Open AWS Lambda Console → Select your function.
2. Click **Configuration** → **Environment Variables**.
3. Add:
   ```
   MINIO_ACCESS_KEY = YOUR_ACCESS_KEY
   MINIO_SECRET_KEY = YOUR_SECRET_KEY
   MINIO_ENDPOINT = objectstore.e2enetworks.net
   MINIO_BUCKET = E2E-BUCKET_NAME
   S3_BUCKET = YOUR-S3-BUCKET-NAME
   ```
4. Click **Save**.

## 6. Configuring S3 Event Notification
1. Navigate to **Amazon S3 Console**.
2. Select the **source bucket**.
3. Click **Properties** → **Event Notifications** → **Create Event Notification**.
4. Configure:
   - **Name**: s3-e2e
   - **Event types**: Check all (Create, Remove, Restore).
   - **Destination**: Choose **Lambda Function** → Select your function.
5. Save changes.

## 7. Testing the Lambda Function
1. Click **Test** → Select the created test event.
2. Click **Invoke**.
3. The function should display `Replication Successful` if it works correctly.

## Limitations & Workaround
⚠ **Existing Data Consideration** – Old or manually uploaded EOS objects may be deleted.
✅ **Workaround**: Upload old EOS folder objects to S3 first to allow tracking.

## Recommendation
Test the solution in a **test environment** before applying it to **production**.

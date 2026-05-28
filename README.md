#PHOTOS_AT_REST
A fully serverless, event driven moderataed photo sharing service fully built on AWS.


Architecture
User
 ↓
Frontend (S3 static site)
 ↓ POST /presign
API Gateway → Lambda (presign) → generates presigned URL → saves metadata to DynamoDB
 ↓ PUT directly to S3
S3 raw-uploads/
 ↓ S3 ObjectCreated trigger
Lambda (moderator)
 ↓ downloads image bytes
Amazon Rekognition — DetectModerationLabels (75% confidence)
 ↙               ↘
SAFE              UNSAFE
 ↓                  ↓
Move to          Delete from S3
approved/
 ↓                  ↓
DynamoDB        DynamoDB
status:approved  status:rejected
 ↓                  ↓
SNS email ✅    SNS email ❌
 ↓
Frontend gallery fetches via GET /images → API Gateway → Lambda → DynamoDB

![API Gateway](screenshots/api.png)


AWS SERVICES USED
S3Image storage (raw-uploads/ + approved/) ap-south-2
Lambda (presign)Generates presigned URL, saves metadata to DynamoDB ap-south-2
Lambda (moderator)S3-triggered, runs Rekognition, moves/deletes image, updates DynamoDB, sends SNS ap-south-2
Lambda (getImages)Fetches all image records from DynamoDB for galleryap-south-2
API GatewayHTTP API — POST /presign, GET /images ap-south-2
Amazon Rekognition Detect Moderation Labels on image bytes ap-south-1
DynamoDB Image metadata storeap-south-2
SNS Email notification on moderation resultap-south-2
IAM Roles and least-privilege policies per Lambda Global
CloudWatchLambda logs and monitoring ap-south-2

![S3 Buckets](screenshots/s3.png)
![S3 Approved](screenshots/s3%20approved.png)
![Lambda Functions](screenshots/lambda.png)
![Presign Lambda](screenshots/presign.png)
![Moderator Lambda](screenshots/mod.png)
![DynamoDB Table](screenshots/dynamo%20db%20table.png)
![SNS](screenshots/sns.png)

Presigned URL pattern
Instead of uploading through API Gateway (10MB limit), the frontend requests a one-time presigned URL from Lambda and uploads directly to S3. This bypasses payload limits and reduces Lambda cost.
Image bytes instead of S3Object reference
Rekognition's S3Object reference only works when Rekognition and S3 are in the same region. Since ap-south-2 doesn't support Rekognition, the moderator Lambda downloads the image bytes and passes them directly — making it region-agnostic.
Recursive trigger prevention
The S3 trigger fires on all PUT events in the bucket. When the moderator copies a file to approved/, it would re-trigger itself. A startswith('raw-uploads/') check at the top of the handler prevents this.
Single DynamoDB write per upload
Metadata is written twice — first as pending by the presign Lambda (so name/email/title are captured immediately), then updated to approved/rejected by the moderator Lambda after Rekognition runs.

Setup
Prerequisites

AWS account with free tier
All resources in ap-south-2 except Rekognition (ap-south-1)

1. S3

Create bucket, turn off Block Public Access
Create folders: raw-uploads/ and approved/
Add CORS config (PUT, GET, AllowedOrigins: *)
Add bucket policy allowing s3:PutObject and s3:GetObject on /*

2. DynamoDB

Create table photosatrest-images, partition key imageid (String)

3. SNS

Create Standard topic photosatrest-notifications
Add email subscription and confirm

4. Lambda

Create 3 functions (Python 3.14)
Attach: AmazonS3FullAccess, AmazonDynamoDBFullAccess, AmazonRekognitionFullAccess, AmazonSNSFullAccess
Add env var SNS_TOPIC_ARN to moderator Lambda
Add S3 trigger on moderator: ObjectCreated:PUT, prefix raw-uploads/
Add resource-based policy on moderator allowing s3.amazonaws.com to invoke

5. API Gateway

Create HTTP API
Routes: POST /presign → presign Lambda, GET /images → getImages Lambda
Add OPTIONS /presign route
Configure CORS: Allow-Origin *, Allow-Methods POST GET OPTIONS
Add resource-based policy on getImages Lambda allowing apigateway.amazonaws.com

6. Frontend

Update API_BASE in index.html with your API Gateway URL
Open in browser or host on S3

![Frontend Gallery](screenshots/front%20end.png)

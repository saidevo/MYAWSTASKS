## Automatically Update RDS Database with Lambda and S3
Lambda function triggers on S3 PUT events in the specified bucket.
Loads CSV files from S3 into the specified RDS database.
Publishes notifications to SNS and sends messages to SQS for each processed file.
## Step 1: Set Up S3 Bucket
Go to the S3 console and create a new bucket.
Note the bucket name as you'll need it later.
## Step 2: Set Up RDS Instance
Go to the RDS console and create a new database instance.
Note the database endpoint, username, and password.
## Step 3: Create Lambda Function
Go to the Lambda console and create a new Lambda function.
Choose a runtime (e.g., Python 3.8).
## Step 4: Configure S3 Bucket Notification
In the S3 console, go to the bucket's properties.
Under "Event notifications," add a new notification:
Choose "All object create events" or specify the suffix .csv.
Select "Send to Lambda Function" and choose the Lambda function you created.
## Step 5: Set IAM Role for Lambda
Go to the IAM console and create a new role.
Attach the following policies to the role:
AmazonS3ReadOnlyAccess
AmazonRDSFullAccess
AWSLambdaBasicExecutionRole
Attach this role to your Lambda function.
## Step 6: Write Lambda Function Code
Below is a sample Lambda function code that reads a CSV file from S3 and updates an RDS database:
```python
import boto3
import pymysql
import os
 
s3_client = boto3.client('s3')
sns_client = boto3.client('sns')
sqs_client = boto3.client('sqs')
 
def lambda_handler(event, context):
    bucket_name = os.environ['BUCKET_NAME']
    rds_endpoint = os.environ['RDS_ENDPOINT']
    db_user = os.environ['DB_USER']
    db_password = os.environ['DB_PASSWORD']
    db_name = os.environ['DB_NAME'].strip()  # Remove any leading/trailing spaces
    sns_topic_arn = os.environ['SNS_TOPIC_ARN']
    sqs_queue_url = os.environ['SQS_QUEUE_URL']

    connection = None
    cursor = None
 
    try:
        connection = pymysql.connect(host=rds_endpoint, user=db_user, password=db_password)
        cursor = connection.cursor()
        # Ensure the database exists
        cursor.execute(f"CREATE DATABASE IF NOT EXISTS {db_name}")
        cursor.execute(f"USE {db_name}")
        cursor.execute("truncate table movies_list")
        # Ensure the table exists
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS movies_list (
                id INT,
                title VARCHAR(100),
                tagline VARCHAR(100)
            )
        """)
        for record in event['Records']:
            s3_object_key = record['s3']['object']['key']
            # Download the file from S3
            download_path = f'/tmp/{s3_object_key}'
            s3_client.download_file(bucket_name, s3_object_key, download_path)
            # Load the data into Aurora MySQL
            with open(download_path, 'r') as file:
                csv_data = file.readlines()
                for row in csv_data:
                    # Assuming CSV format: id,title,tagline
                    row_data = row.strip().split(',')
                    cursor.execute(
                        "INSERT INTO movies_list (id, title, tagline) VALUES (%s, %s, %s)",
                        (row_data[0], row_data[1], row_data[2])
                    )
            connection.commit()
            # Send notification to SNS
            sns_client.publish(
                TopicArn=sns_topic_arn,
                Message=f"File {s3_object_key} has been processed and loaded into RDS."
            )
            # Send message to SQS
            sqs_client.send_message(
                QueueUrl=sqs_queue_url,
                MessageBody=f"File {s3_object_key} processed."
            )
    except pymysql.MySQLError as e:
        print(f"Error: {e}")
        raise e
    finally:
        if cursor:
            cursor.close()
        if connection:
            connection.close()
    return {
        'statusCode': 200,
        'body': 'File processed and loaded into RDS successfully'
    }
```

## Step 7: Set Environment Variables for Lambda
Go to the Lambda function's configuration.
Under "Environment variables," set the following variables:
RDS_HOST = db-lambda-trigger.cluster-cf8mso6wgybr.us-east-1.rds.amazonaws.com
RDS_USERNAME = admin
RDS_PASSWORD = admin123
RDS_DB_NAME =
## Step 8: Test the Setup
Upload a CSV file to the S3 bucket:

Use the S3 bucket named lambda-trigger-for-rds.
Ensure the CSV file is in the correct format for processing.
Check the Lambda function logs:

Go to the AWS Lambda console.
Navigate to your Lambda function.
Check the logs in Amazon CloudWatch to ensure the function is triggered and processing the file.
add configure test event for lambda function
```
 {
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "awsRegion": "us-east-1",
      "eventTime": "2024-06-27T00:00:00.000Z",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": {
          "name": "s3_rds"
        },
        "object": {
          "key": "organization.csv"
        }
      }
    }
  ]
}
```

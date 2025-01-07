# AWS Lambda Data Processing Pipeline.
The project below demonstrates how to use serverless technologies to process data in real time. The pipeline is designed to handle file uploads, extract meaningful insights from the data and and save the results back to S3, all without manual intervention.

# Overview
The pipeline works as follows:

1. A CSV file is uploaded to an S3 Bucket.
2. AWS Lambda is triggered automatically by the file upload event.
3. The Lambda function processes the file by:
   
     * Calculating summary statistics of the data in the file.
     * The summary file is generated with the calculated statistics
     * The summary file is saved back to the S3 bucket

# Steps to use
 * Create an S3 Bucket
   
   ![23d3b2e1-1b8b-4562-92f1-6b74456eaa13](https://github.com/user-attachments/assets/44c7a4f5-ce1a-4d39-b766-9f54680a2463)

* Create an aws lambda function that is to be triggered when one uploads the file on s3
  ```python
  import boto3
  import csv

  s3 = boto3.client('s3')

  def lambda_handler(event, context):
    # Get the bucket name and file key from the event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']

    # Download the file from S3
    file = '/tmp/data.csv'
    s3.download_file(bucket, key, file)

    # Read the CSV file and calculate statistics
    values = []

    with open(file, 'r') as f:
        # reads the csv file
        reader = csv.DictReader(f)
        for row in reader:
            if row['Amount']:  # Ensure the 'Amount' column exists and is not empty
                values.append(float(row['Amount']))

    # Calculate the total and mean
    total = sum(values)
    mean = total / len(values) if values else 0

    # Print the results
    print('Total Amount:', total)
    print('Mean Amount:', mean)

    # Create a summary
    summary = f"Total Amount : {total}\nMean Amount: {mean}"

    # Save the summary to a text file
    summary_file = '/tmp/summary.txt'
    with open(summary_file, 'w') as f:
        f.write(summary)

    # Upload the summary file back to S3
    summary_key = 'emma-processed-files/' + key.replace('.csv', '_summary.txt')
    s3.upload_file(summary_file, bucket, summary_key)

    return {
        'statusCode': 200,
        'body': {
            'file': summary_key,
            'total': total,
            'mean': mean
        }
    }

* Create an event structure and add json command to it to ensure the function receives the data it needs
  ```json
    {
    "Records": [
      {
        "s3": {
          "bucket": {
            "name": "your-bucket-name"
          },
          "object": {
            "key": "your-file-key"
          }
        }
      }
    ]
  }

* Give the lambda role permission to access your S3 bucket by allowing it to retrieve and upload files.
 - Go to **IAM** service and select **Roles**
 - Click the role that was created automatically
 - Proceed to **Permissions** and select **Edit** to update the policy with the code below

   ```json
        {
            "Effect": "Allow",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::emma-datapipeline-bucket/*"
        },
        {
            "Effect": "Allow",
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::emma-datapipeline-bucket/emma-processed-files/*"

- Return to the **Lambda** service and select your function.
- Click **Add trigger**
- Select **S3** as the trigger source.
- In the **Bucket** tab, choose the S3 bucket you created and click **Add** to confirm the trigger setup
  
- This ensures that once an upload has been made to s3 bucket, the lambda function processes the data and returns the results. This happens automatically.

  ![4a76e284-133d-4982-9a61-fdb3d1839ce8](https://github.com/user-attachments/assets/51fc088c-16ff-4b24-96ae-41377bb34d36)

  

* Testing the function
  - Go back to the s3 bucket you created.
  - Upload a file that you want to test with.
  - Check if the lambda function gets triggered and processes the file
  - You should receive the summary of your data as a txt file.

  ![4e347136-698c-41c0-8639-a78119d9cfb2](https://github.com/user-attachments/assets/5e9c8832-2d7e-46d1-b054-cff683d6861d)

## RESULTS
  
  ![5df2bc8d-68d4-4ace-a5db-f919261b2fe0](https://github.com/user-attachments/assets/ea016fa0-7faa-4716-9426-b3760ea567e4)

# Configure Cloud Storage S3 and Permissions

The integration between Snowflake and an S3 bucket using Snowpipe is a way to automatically and continuously load data from AWS cloud storage into the Snowflake database. Snowpipe utilizes S3 event notifications to trigger the data loading process in micro-batches, eliminating the need to schedule or manually execute COPY commands. To set up this integration, follow these steps:


## Create AWS S3 Bucket

1. Sign in to the AWS Management Console and open the Amazon S3 console.
2. In the left navigation pane, choose **Buckets** and than choose **Create bucket**.
3. Enter a name for your bucket.
4. For **Region**, choose the AWS Region where you want the bucket to reside.
5. Under **Block Public Access settings for this bucket**, choose the Block Public Access settings that you want to apply to the bucket.
6. Choose Create bucket.

You've created a bucket in Amazon S3.

## Create IAM Policy for Snowflake's S3 Access

Snowflake needs IAM policy permission to access your S3 with `GetObject`, `GetObjectVersion`, and `ListBucket`. Log into
your AWS console and navigate to **Policies** and use the JSON below to create a new IAM policy named *snowflake_access*.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
              "s3:PutObject",
              "s3:GetObject",
              "s3:GetObjectVersion",
              "s3:DeleteObject",
              "s3:DeleteObjectVersion"
            ],
            "Resource": "arn:aws:s3:::<bucket_name>/<prefix>/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ListBucket",
                "s3:GetBucketLocation"
            ],
            "Resource": "arn:aws:s3:::<bucket_name>",
            "Condition": {
                "StringLike": {
                    "s3:prefix": [
                        "<prefix>/*"
                    ]
                }
            }
        }
    ]
}
```
>Make sure to replace `bucket` and `prefix` with your actual bucket name and folder path prefix.

## New IAM Role

On the AWS IAM console, add a new IAM role tied to the *snowflake_access* IAM policy. 
Create the role with the following settings.

- Trusted Entity: Another AWS account
- Account ID: <your_account_id>
- Require External ID: [x]
- External ID: 0000
- Role Name: snowflake_role
- Role Description: Snowflake role for access to S3
- Policies: snowflake_access

Create the role, then click to see the role's summary and record the Role ARN.

## Integrate IAM user with Snowflake storage.

Within your Snowflake web console, you'll run a `CREATE STORAGE INTEGRATION` command on a worksheet.

```snowflake
CREATE OR REPLACE STORAGE INTEGRATION S3_role_integration
  TYPE = EXTERNAL_STAGE
  STORAGE_PROVIDER = S3
  ENABLED = TRUE
  STORAGE_AWS_ROLE_ARN = "arn:aws:iam::<role_account_id>:role/snowflake_role"
  STORAGE_ALLOWED_LOCATIONS = ("s3://<bucket>/<path>/");
```
>Be sure to change the `<bucket>`, `<path>` and `<role_account_id>` is replaced with your AWS S3 bucket name, folder 
>path prefix, and IAM role account ID.

### Run storage integration description command

Run the command to display your new integration's description.

```snowflake
desc integration S3_role_integration;
```
Record the property values displayed for `STORAGE_AWS_IAM_USER_ARN` and `STORAGE_AWS_EXTERNAL_ID`

## IAM User Permissions

Navigate back to your AWS IAM service console. Within the **Roles**, click the *snowflake_role*. On the 
**Trust relationships** tab, click **Edit trust policy** and edit the file with the `STORAGE_AWS_IAM_USER_ARN` and 
`STORAGE_AWS_EXTERNAL_ID` retrieved in the previous step.

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "<STORAGE_AWS_IAM_USER_ARN>"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "<STORAGE_AWS_EXTERNAL_ID>"
        }
      }
    }
  ]
}
```
Click **Update Policy**


# Create a pipe in Snowflake

Now that your AWS and Snowflake accounts have the right security conditions, complete Snowpipe 
setup with S3 event notifications.

### Create a Database, Table, Stage, and Pipe
On a fresh Snowflake web console worksheet, use the commands below to create the objects needed for Snowpipe ingestion.

#### Create Database

```snowflake
create or replace database S3_db;
```
Execute the above command to create a database called **S3_db**.

#### Create Stage

```snowflake
use schema S3_db.public;
create or replace stage S3_stage
  url = ('s3://<bucket>/<path>/')
  storage_integration = S3_role_integration;
```
To make the external stage needed for our S3 bucket, use this command. 
Be mindful to replace the `<bucket>` and `<path>` with your S3 bucket name and file path.

#### Create Table

This command will make a table by the name of ‘S3_table' on the S3_db database.

```snowflake
create or replace table S3_table(files string);
```

#### Create Pipe

The magic of automation is in the create pipe parameter `auto_ingest=true`. With auto_ingest set to true, data stagged 
will automatically integrate into your database.

```snowflake
create or replace pipe S3_db.public.S3_pipe auto_ingest=true as
  copy into S3_db.public.S3_table
  from @S3_db.public.S3_stage;
```

```Snowflake
show pipes;
```

### Check the Pipe status and copy the "notificationChannelName" for the AWS SQS

```snowflake
select SYSTEM$PIPE_STATUS('S3_db.public.S3_pipe');
```

### Creating a New S3 Event Notification to Automate Snowpipe

Sign in to the AWS Management Console and open the Amazon S3 console. In the **Buckets** list, choose the name of the 
bucket that you want to enable events for. Choose **Properties** and navigate to the **Event Notifications** section and 
choose **Create event notification.**

Complete the fields as follows:

- Name: Name of the event notification (e.g. Auto-ingest Snowflake).
- Event Types: Select **All object create events**.
- Destination: Select **SQS Queue**.
- Specify SQS queue: Select **Enter SQS queue ARN**.
- SQS queue ARN: Paste the **"notificationChannelName"** from the SHOW PIPES output.

Choose **Save changes**, and Amazon S3 sends a test message to the event notification destination.

Check the Pipe status to check if the pipe is running.

```snowflake
select SYSTEM$PIPE_STATUS('S3_db.public.S3_pipe');
```
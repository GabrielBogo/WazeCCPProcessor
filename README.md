# WazeCCPProcessor

Takes [Waze CCP](https://www.waze.com/ccp) data feed and processes it into a cloud database for querying, analysis, API hooks, and mapping.

## Overview

Louisville is creating an automated cloud processing solution that can be replicated by any CCP Partner, with the help of other govs, partners, and sponsors.

You grab this [Terraform.io](http://www.terraform.io) code and deploy the infrastructure stack (currently AWS but cloud agnostic).

You enter your CCP data feed URL as a parameter.

Then you can store, analyze, query, extract live and historic data for your city.

See the [Projects](https://github.com/LouisvilleMetro/WazeCCPProcessor/projects) area for how you can help, and the [Wiki](https://github.com/LouisvilleMetro/WazeCCPProcessor/wiki) for all the details.

## Deploy the Solution to Your Cloud

We have an end-to-end data processor and database working that you can deploy.  It saves your CCP data as JSON files every 2 minutes, and processes the data into a combined real-time and historic database.

**Here are the steps to make it work:**

### AWS setup

1. Log into your own AWS console.

**Create IAM User**
1. Go to IAM, Add User, name `waze_terraform_user`. Chose programmatic access, attach policy directly: Administrator Access. Create User.
2. Copy access key and secret access key

*Note: when finished deploying the code, remove this admin user for security. We could build a security policy for this in the future.*

**Create State Management Bucket** (one-time)

1. Go to [S3](https://s3.console.aws.amazon.com/s3/home), create bucket `waze-terraform-state-management-CITYNAME`, default properties and permissions, create.
2. Choose a region that has all services needed. Note your region for later.

### Download Git Repo Code and Configure

1. Download this repo to a folder on your computer.
1. On your desktop, go to `/infrastructure/terraform/backend/config` and edit the file.  Add the name of your state management bucket `waze-terraform-state-management-CITYNAME`, and region (eg. "us-east-1").
1. Go to `/infrastructure/terraform/modules/globals/globals.tf` and update the following:
```
# region where resources will be created by default
output "default_resource_region" { value = "us-east-1" }

output "waze_data_url" { value = "YOUR SPECIFIC WAZE DATA FULL HTTP URL HERE" }
output "rds_master_username" { value = "YOUR DESIRED DB USER NAME HERE" }
output "rds_master_password" { value = "YOUR DESIRED DB ADMIN PASSWORD HERE" }
output "lambda_db_password" { value = "YOUR DESIRED PASSWORD FOR THE LAMBDA PROCESSING USER HERE"}
```

### Setup Terraform for the First Time (one-time)

1. Download the [latest version](https://www.terraform.io/downloads.html) v0.11, unzip, move to /bin if needed, and [set the path](https://www.terraform.io/intro/getting-started/install.html) (eg, sudo ln -s terraform terraform).
1. Verify your version is correct by running `terraform --version`.

### Running Terraform
1. In your terminal go to your `/infrastructure/terraform/environment/env-dev` directory (use `env-dev` for a development/test deployment, or `env-prod` for production level deployment)
    - The dev environment has options set that will allow destroying everything with no extra work other than `terraform destroy`
    - The prod environment has options set that will prevent destroying the database unless you have taken a final snapshot and will prevent destroying the S3 buckets if there are files in them.  This is done to help protect production data from accidental, unrecoverable destruction.
1. Set session variables for Terraform with access keys from AWS IAM user:
    - `export AWS_ACCESS_KEY_ID="YOUR IAM USER ACCESS KEY"`
    - `export AWS_SECRET_ACCESS_KEY="YOUR IAM USER SECRET ACCESS KEY"`
1. Run the following commands
    - `terraform get`
    - `terraform init -backend-config="../../backend/config"`
    - `terraform plan`
    - `terraform apply`
1. Make a note of `db_cluster_endpoint` value that will be output when Terraform completes.

### Running SQL schema creation script
After the stack is up and running use you favorite PostGres connection client (eg, DBeaver, pgAdmin) and connect to the `db_cluster_endpoint` from above using your previously specified `rds_master_username` and `rds_master_password`.

1. Open `/code/sql/schema.sql`
1. Update the password for the `lambda_role` (near the top) to match what you provided in the terraform config
1. Connect to the DB using the `db_cluster_endpoint` value output from Terraform, the DB schema `waze_data`, and your DB admin username and password from file configurations.
1. Run script in your DB client

*Note: this is a manual process for now to ensure DB updates are applied accurately for the moment.*

### Using the optional SNS notifications

The system makes use of several SNS topics that can optionally be subscribed to in order to receive notifications or trigger other events.  Currently there are 4 topics available:
  - *File received:* notification that fires every time a file is added to the incoming S3 bucket
  - *File processed:* notification that fires when we've finished processing a file; this is optional sending notifications to this topic can be disabled in terraform
  - *Records in dead-letter queue:* sends notifications when records are in the dead-letter queue; this is optional sending notifications to this topic can be disabled in terraform
  - *Records in work queue:* notification that fires if there are records in the work queue that need to be processed; because of the nature of the queue, there may not _actually_ be anything left in the queue when the notification is sent

Example usages of these topics:
  - Subscribe with email to the "Records in dead-letter queue" topic to get an email when things go to that queue (usually indicates errors)
  - Subscribe a web hook to the "File processed" notification to kick off an external process that needs to read the new data from the database

The ARNs for each of the topics can be found in the outputs after running terraform, and you can view them in the AWS Console.

### Clean Up

Go to your AWS IAM area and delete the `waze_terraform_user` you created.

## Finished Result

This creates an infrastructure stack which has pings your custom Waze CCP data feed every 2 minutes and save the JSON to a new bucket, which then gets processed into the relational database.  There is error handling and also notification options for when things go right or wrong.  

Here's what was created:

![Waze Current Architecture](docs/Current%20Architecture.png "Waze Current Architecture")

You can update the stack with new infrastructure as the code here gets updated, and it only affects new and changed items. You can also remove all the infrastructure automatically (minus the S3 bucket you created manually) by deleting the Terraform stack using `terraform destroy` after the `get` and `init` commands. 

## Loading Historic JSON Data Files

You can also dump any previously collected historic JSON files into your bucket and the processor will go through them and save/update the relevant data into your database.  Using `aws s3 sync` is a good place to start to copy files in chunks from a previous bucket to a new bucket.  

### Notes on processing many files at once

The system will queue up and process every file that gets added to the incoming data bucket.  This makes it easy to process any old files you may have already collected, or reprocess files later if changes are made that would require it.  If you should decide to dump a mass of files in the bucket, you may want to consider temporarily disabling all of the foreign keys.  Doing so will _greatly_ increase throughput, which also means reduced cost to run.  Disabling the foreign keys is not without risk, though, so it is advisable to create a backup beforehand and understand what you might need to do to clean up should inconsistent data get loaded while the keys are off.  We are working on a script you can run to disable and re-enable your FKs. 

### Dealing with Problem Files

If you have JSON files sitting in your `development-tf-waze-data-incoming-***` bucket for an hour or more that means there was an error processing them.  The errors will show up in your [SQS Queue](https://console.aws.amazon.com/sqs/home) called `development-tf-waze-data-processing-dlq` if you want more details.  

It could be they were saved before you ran your database script.  In that case, go to your `incoming` bucket, checked all files, select Open, they will all download to your desktop, click Upload, selected those same files, upload them and they will now all process and be moved to `processed`.

It could be that there are other errors with the file, like a schema change or network issue, etc.  In that case open an Issue in this repo and upload the file and we'll try to see what's wrong and maybe improve the process/code.

## Costs

This config stands up infrastructure that is mostly cheap/free (depending on usage), but the database itself is pretty powerful and will result in monthly charges in excess of $200 (as of this writing).  We are working on ways to reduce the costs and you can help out on this [issue](https://github.com/LouisvilleMetro/WazeCCPProcessor/issues/32).

## Current Plans

We are working on writing API hooks, data visualizations and tools, and maps, which is all part of our project roadmap.

See our [Projects](https://github.com/LouisvilleMetro/WazeCCPProcessor/projects) area for our blueprint of how we are proceeding. 

**We would like to collaborate with you!**  Please suggest updates, work on the [help wanted issues](https://github.com/LouisvilleMetro/WazeCCPProcessor/issues?q=is%3Aissue+is%3Aopen+label%3A%22help+wanted%22), collaborate on the Wiki, etc.  It would be great to work together to get the best solution, use cases, and finish faster.   

We've build out the code in [Terraform](http://www.terraform.io) and supported AWS at first, but would like it to be deployed to any cloud provider.  See our [Issues](https://github.com/LouisvilleMetro/WazeCCPProcessor/issues) area for how you can help with this.

## Background

If you'd like a little more background on Louisville and what our city has been going with Waze and other mobility data, take a look at these links:

1. [Louisville Waze Internal Hackathon Recap](https://medium.com/louisville-metro-opi2/waze-louisvilles-first-internal-hackathon-647363a85392)
2. [Harvard Civic Analytics Network Presentation - Slides](https://docs.google.com/presentation/d/1esPVvhuIRjD199rN8aimK_XcmCt0pJOkjEIyCMhGKks/)
3. [Waze April 2018 Monthly Call - Slides](https://docs.google.com/presentation/d/1loAV4BDAUyXdrn44QoLmYiwZdLmL59C4jvJGlZ1a-AY/)
4. [Open Government Coalition](https://www.govintheopen.com/)

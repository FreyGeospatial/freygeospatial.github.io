---
layout: post
title:  How to Connect an R Shiny Dashboard to AWS S3
categories: [R, Business Intelligence, AWS]
---

The applications for hosting an R Shiny web application on the cloud are huge, and learning to work with cloud services in general is critical for data analysts, engineers, and scientists. Dashboards that rely on cloud-hosted data do not need to be constantly redeployed; as long they point to the right file on S3 (or Redshift or RDS database), your Shiny application will use the most recently available data. This has the potential to reduce overhead and provide near-real time access to information for your clients.

In this post, I will demonstrate how to:
1) Set up an Amazon Web Services (AWS) S3 Bucket
3) Connect S3 to an R script
4) Use this connection in an R Shiny dashboard.

## Setting up an Amazon S3 Bucket

**Create an AWS Account**. You will be asked for an email, password, and credit card information. As long as you stay within Free Tier restrictions, you will not be charged. Only my use of the AWS Key Management Service (KMS) has incurred charges from the workflow described in this tutorial, but currently I've only been charged $0.12.

**Create an S3 Bucket**. The name for your bucket must be unique- no other AWS user must have created a bucket with the same name.

**Choose an applicable region**. The "cloud" just refers to data centers that Amazon maintains.  These data centers are located in various regions. I chose the region closest to me as of the writing of this tutorial. Ensure that you remember the region you picked; transferring data between regions can incur costs, so if you have other AWS services that you anticipate needing in the future, having them located in one region will reduce spending.

<div style="display: flex;">
    <div style="flex: 80%; text-align: center;">
        <img src="/images/r-shiny-aws/createbucket_orig.png" alt="create s3 bucket">
    </div>
</div>

<br>

**Block all public access to this bucket**. Amazon recommends this security feature be activated to prevent unrestricted access to the contents of your bucket.

<div style="display: flex;">
    <div style="flex: 80%; text-align: center;">
        <img src="/images/r-shiny-aws/blockaccess_orig.png" alt="create keys">
    </div>
</div>

<br>

**Encrypt your data**. I can't stress the importance of data security enough. I recommend you enable server-side encryption. This will encrypt all the data at the object level that enters the bucket. You have the option of using either SSE-S3 encryption of SSE-KMS (Key Management Service) encryption. It is up to you, however, to decide what is more suitable for your needs. I personally chose to use SSE-KMS, with symmetric encryption. I then assigned a specific IAM user to have access to the key. (For details on setting up an IAM user, see the next step; this can be done concurrently).

When you use SSE-KMS, you create an *Access Key ID*, *Secret Access Key*, and *user* for said key. Ensure you keep these credentials safe- you will need these for APIs and other services (e.g, R Shiny).

<div style="display: flex;">
    <div style="flex: 80%; text-align: center;">
        <img src="/images/r-shiny-aws/kms_orig.png" alt="create keys">
    </div>
</div>

<br>

<b>Set-up IAM users:</b> IAM stands for Identity and Access Management. This allows you to enable users to access certain AWS services without needing access to the root account. This improves security and reduces risk of unauthorized access to data and cloud services.

​<b>Complete the set-up of your bucket and upload files.</b> Feel free to upload some test files. I decided to upload an RDS file for ease of use with R. This RDS file contains a simple Leaflet map I created previously that I wanted to be able to showcase easily on Shinyapps.io. I ensured all my read/write permissions for the file were set to 'private'.

<b>​Optional: Set up budget alerts.</b> In the management console under billing, you have the option to set up email and SNS alerts if you begin to go over budget. I have mine set to $10/month, and will receive notifications if I approach 80% of that amount.


### Connect R to your S3 Bucket

Now comes the easy part! Seriously- connecting R to S3 is a piece of cake, if you are already an R programmer. First, store your S3 credentials in a file named "Renviron" in your working directory. *While it is not good practice to hard code credentials*, this should get your code up and running for at least a proof of concept and demo. <b>Just don't forget to pass these credentials in using a safer method before putting it into production!</b> 

The format for saving your credentials in the Renviron file should be as follows:

```
AWS_ACCESS_KEY_ID = "XXXXXXXXXXX"
AWS_SECRET_ACCESS_KEY = "XXXXXXXXX"
AWS_DEFAULT_REGION = "XXXXXXX"
```

R will know where to find these credentials. I've also hidden the file names and paths within text files located in my working directory for the same reason. Run my code below (albeit with the correct folder paths):

```R
library(aws.s3)

s3BucketName <- scan("bucketname.txt", what = "txt")
s3File <- scan("filepath.txt", what = "txt")

# These files are located on the bucket
file_names <- get_bucket_df(s3BucketName)

# This loads my desired file - in my case, a Leaflet map stored in an RDS file.
myMap <- s3readRDS(object = file_names[file_names$Key == s3File, "Key"], bucket = s3BucketName)

myMap
```

Implementing this into a ​Shiny dashboard is relatively straightforward once you've completed the previous task. We simply need to wrap all this into a new script called app.R.


```R
library(shiny)
library(shinydashboard)
library(leaflet)
library(aws.s3)

s3BucketName <- scan("bucketname.txt", what = "txt")
s3File <- scan("filepath.txt", what = "txt")

# files in bucket
file_names <- get_bucket_df(s3BucketName)
myMap <- s3readRDS(object = file_names[file_names$Key == s3File, "Key"], bucket = s3BucketName)

ui <- dashboardPage(
    dashboardHeader(),
    dashboardSidebar(),
    dashboardBody(
        leafletOutput("mymap", height = "92vh") #this text is an ID that must match `output$var_name` below***
    )
)

server <- function(input, output, session) {
    
    output$mymap <- renderLeaflet({ #****
        myMap
    })
}

shinyApp(ui, server)
```

<br>

This is a pretty bare-bones dashboard- merely a structure for you to build off of. But I hope you've found this tutorial helpful for merging these technologies together. And once you build out the the dashboard, deploying it online is pretty simple using [Shinyapps.io](Shinyapps.io){:target="_blank"}.

And again, don't forget that after your demo and POC, you will want to pull in your AWS credentials from a secrets store, whether that is with GitHub Secrets, AWS Parameter Store, or AWS Secrets Manager. 

If you decide to host your application with AWS EC2 to host the application, you might decide to use [instance profiles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html){:target="_blank"} as a method of authenticating your application against your cloud provider.

Thanks for reading, and as always, happy coding!
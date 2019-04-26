# CloudFormation deployment scripts - Sun Orchrad Freight App

These scripts achieve two main goals - they simplify deployment of the app and allow for smooth CI/CD from GitHub when source code changes. It also allows you to deploy multiple independent instances of the app, e.g. for testing and production. You can find information on the app architecture  and deployment of the app below.

## App architecture

The app is designed to use strictly Serverless resources that can be deployed to the AWS environment. The AWS technologies and services used directly by the app are:

* S3 - used for hosting the frontend app and for data import. Also, S3 buckets are created by some of the deployment services.
* Cloud Front - used for serving the frontend app and backend API to the users.
* Route 53 - stores DNS records for the app.
* Lambda - runs the backend and data import functions.
* Api Gateway - publishes the backend Lambda function as a REST API.
* Cognito - used for authentication and authorization of the app (both frontend and backend).
* Aurora Serverless - stores the data in MySQL compatible database. The database server is activated only when used, significantly reducing the cost of the app operation.
* VPC - used for connectiong the backend Lambda functions to the Aurora database.

There are also other technologies used for app deployment:

* Cloud Formation - used for deploying the app to the AWS account.
* Code Pipeline - used for CI/CD from GitHub.
* Code Build - used for building the components of the app in the AWS environment when code in GitHub changes.
* IAM - used to authorize the above services to make changes to the AWS aenvironment (e.g. create resources).

The app consists of four main parts:

### Frontend

Frontend app is a responsive Javascript Single-Page Application (SPA). The app is hosted in AWS S3 and served via CloudFront cache to allow HTTPs connection. The frontend app connects to the backend API using REST. Following technologies are used:

* Vue.js - Javascript SPA framework
* Vue.js modules - Vuex (state store), Vuetify (UI), Vue-reource (REST)
* Cognito authentication -  AWS Aplify Javascript SDK (`amazon-cognito-identity-js`)
* Webpack - Dependency and build too for Javascript apps
* Babel - Poly-fill to enable support for different browser versions
* SPA Prerender - pre-rendering the app during build improves performance and SEO

### Backend

The backend is implemented as a Javascript/Node.js application that runs in AWS Lambda. The backend defines a REST API that is accesible via the API Gateway service. The backend connects to the Aurora Servereless relational database (MySQL compatible). Following technologies are used:

* Node.js.
* Express.js - framework for creating web server/API in Node.js.
* `aws-serverless-express` module that allows Express app to run in Lambda.
* MySQL nativeJavascript client  - `mysql` npm module. 

### Import script

The import script runs in AWS Lambda. It is activated by a file upload to a given S3 bucket. The Lambda function reads the file from the S3 bucket and processes it. There are two file types supported (distinguished by file extension):

* `.csv` file - this file is expected to by an export from the `velocity` database. The records are processed and imported to the app as new BOLs that can be assigned to loads. If the data import references unknown customer, broker, distributor, region, or route, these are also created in the app.
* `.sql` file - this file is sent directly to the database engine. It must contain SQL commands. It is intended for database schema changes or batch import (e.g. to import existing distributors, brokers, customers,...)

The data import should take only a few minutes. You can then delete the file from the S3 bucket.

### Database

The database is deployed as a Aurora Serverless database. The database engine has no public endpoint and can be accessed only through the VPC. For this reason, the deployment template creates a dedicated VPC for the app that allows for communication wit the database service.

## Deployment and CI/CD

The CloudFormation template creates three separate CodePipelines and CodeBuild projects for the CI/CD of the app from GitHub repos.

The pipeline from the frontend deploys a Javascript Single-Page-App to S3 bucket, the app is served by CloudFront and there is an A record created in the Route53 zone to link to the CloudFront distribution.

The pipeline for the backend deploys a lambda function that connects to the Aurora Serverless DB. The lambda function is accessible through the API Gateway and authenticated using AWS Cognito service.

The third pipeline is responsible for the import routine. It deploys a Lambda function that reacts on S3 events (creating an object) and imports the data to the DB.

The Lambda functions connect to the Aurora Database using a custom VPC that is also created by the CloudFormation. The VPC uses S3 Gateway Endpoint so no NAT Gateway is used.

## Prerequisities

Before deploying the app there are several steps that must be completed so all the required information can be provided to the Cloud Formation.

 1. You need to generate a new access token for your GitHub account that has access to the private app repos. The access token is required to enable CI/CD pipeline. Follow the [instructions on GitHub)(<https://help.github.com/en/articles/creating-a-personal-access-token-for-the-command-line>) to create the access token. The access token should be created for the `actaeonsys` account that has rights to access the app repositories. It is also possible to create a completely new GitHub user, give him rights to the repos and create the access token for this new user if you wish.
 1. There must be a Route 53 Hosted Zone available in the AWS environment where the DNS records for the app deployment will be placed. If there is no such hosted zone, the template has to be adjusted and the DNS records must be created manually in your custom DNS registry.
 1. You need to have the ARN of the HTTPS certificate that can be used with the newly created DNS records. You can issue the certificate using the [AWS Certificate Manager (ACM)](<https://console.aws.amazon.com/acm>) In my case, I have the hosted zone for the domain `nastojte.cz` and one certificate issued for `*.nastojte.cz`, so this certificat can be used for all the sub-domains. If you have such certificate with the wildcard `*` issued, you can use this one. Otherwise, you have to create the the certificates for the subdomains you intent to use prior the deployment. After the certificate is issued, you can copy its ARN from the certificate details view in AWS console.

## Deployment

To deploy the app to the AWS, sign in to the console as a user with `AdministratorAccess` rights and continue to the [Cloud Formation console](<https://console.aws.amazon.com/cloudformation/>). Then create a new stack and provide the S3 URL of the CloudFormation template that you can find below. Next, you will be asked to fill in the template parameters.

### Template link

The template link for the customized Sun Orchard Freight App is <https://s3.eu-central-1.amazonaws.com/akucera-cf-templates/sun-orchard/master.yml>

### Parameters

The template contains a number of parameters you are required to fill in. For most of them, there are meaningful defaults preconfigured, but you can change them if you wish. SOme of them you have to provide by yourself (see Prerequisities above). First of all, you have to provide the **stack name**. This can be any name you wish. The other parameters are:

GitHub settings:

The GitHub setting are preconfigured to connect to my GitHub account to download source code from my repositories to which you should have access (I granted the rights to the `actaeonsys` account). You can also clone the repos and change the configuration accordingly if you wish. You only need to provide the access token.

* **GitHubKey** - Your access token you generated in the Prerequisities step 1.
* GitHubUser - Username of the REPO OWNER.
* GitHubBranch - The branch that contains the source code to be deployed.
* GitHubFrontendRepo - Repository for the frontend app.
* GitHubBackendRepo  - Repository for the backend API.
* GitHubImportRepo - Repository for the import script.

Network settings:

This section requires you to fill several network settings that are unique to your domain and needs. You can change these as you wish.

* RootDomain - The name of the domain (Hosted Zone in Route 53 - see Prerequisities step 2). If the app URL is `sunorchard.nastojte.cz`,the root domain is `nastojte.cz`.
* FrontendSubDomain - The subdomain of the frontend. Prefilled to `sunorchard`.
* APISubDomain - The subdomain of the backend. Prefilled to `sunorchard-api`.
* **FrontendCrtArn** - The ARN of the certificate for the frontend subdomain (see Prerequisities step 3).
* **BackendCrtArn** -  The ARN of the certificate for the backend subdomain (see Prerequisities step 3).

DB configuration:

You are required to configure the database credentials in this section. Most of the parameters is pre-filled wit reasonable defaults, but you have to provide the database password.

* AuroraDBName - Name of the database. Prefilled, no need to change.
* AuroraDBUser  - Name of the root database user. Prefilled, no need to change.
* **AuroraDBPassword** - Password for the DB user. You have to set this, but the database cannot be accessed from the internet so the strength of the password doesn't really matter.
* DataBucketName - Name of the bucket you will import the data from. The bucket **will be created** during the deployment. If the bucket already exists, the deployment fails. Unfortunatelly, the name of the bucket mus be unique in the **whole** S3, not just in your acount or even regon. The prefilled value should be available for use.
* AwakeInterval - Sets the period of inactivity after which the database engine goest to sleep and stops incurring cost. Default is 30 minutes. This can be changed later.

### Executing the template

After you fill in the parameters, you will have to go through several screens with different options. The important ones are the Tags you can add to the created resources. You can use tags to track your costs in the billing console. In the last steps, you will have to confirm that you allow the CloudFormation to create new IAm roles.

Then the deployment executes. You can monitor the progress in the Cloud Formation console. There will be several stacks created in the cloud formation console. After a while, two more stack will apear, their names ending in `ci-cd-stack` (it should not be more than 30 minutes). After these stacks are created, the app is fully deployed.

You can continue to the app configuration steps.

## Next steps

### Create users

Go to the [Amazon Conginto Console](<https://console.aws.amazon.com/cognito>) and select `Manage User Pools`. Find the pool for the app and add new users. The username should be their e-mail address. The different configuration options (send invite, verified e-mail,...) doesn't really matter.

### Import the data to the database

Optionally, you can import the existing brokers, distributors, customers and trucking companies to the app. To do so, you have to download the [SQL script](<https://github.com/oookoook/sun-orchard-import/blob/master/db-schema/enums.sql>) (This links to the private repo - the `acteonsys` user has rights to access it. I can send the file via e-mail is needed). I recommend this so you don't have to enter the trucking companies manually. Also, importing the brokers this way retains their IDs from the Access app.

Upload the sql file to the S3 bucket you sepcified during the deployment. The file extension must be `.sql` (lowercase).

Since this will be the first access to the database, the database schema will be created now. After a few minutes, the data should be imported 9you can watch the logs of the Lambda function from the AWS Lambda console).

Then, you can upload the CSV with the data records from the `velocity` database to the S3 bucket. The data will be imported in a few minutes.

### Logging in to the app

Using your browser, navigate to the URL you specified for the app during the deployment (e.g. <https://sunorchard.actaeonconsulting.com>). Enter the credentials you specified in the Cognito console. The app will require you to change the password. Enter the new password and proceed to the app. If you select a correct region and date, you should be able to view the imported BOLs.

## Deleting the app from the AWS

If you wish to delete the app from the AWS, there are three steps required that must be performed in this exact order:

1. Delete both the stacks ending in `ci-cd-stack`.
1. Delete the main (root) stack (it will also delete all the nested stacks).
1. Delete the S3 buckets that the app used. They can't be deleted automatically by Cloud Formation because they are usually not nempty.

## Resources

This is a list of additonal resources (mostly from AWS Docs that describe the architecture and deployment process in more detail):

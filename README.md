# nodejs-mysql-cloudformation

An example of how you can perform Infrastructure-As-Code (IaC) using AWS CloudFormation and Continuous-Integration/Continuous-Deployment (CI/CD) using AWS CodePipeline.

We will be deploying the NodeJS-ExpressJS-MySQL Create-Read-Update-Delete (CRUD) application at [DigiPie/nodejs-mysql-aws-beanstalk](https://github.com/DigiPie/nodejs-mysql-aws-beanstalk), by using the CloudFormation IaC templates in this repository and setting up a simple AWS CodePipeline.

## Content

- [Architecture](#architecture)
- [Getting started](#getting-started)
- [Continuous-Deployment](#continuous-deployment)
- [References](#references)

## Architecture

![Architecture diagram](architecture_diagram.png "Architecture diagram")

_Figure 1: Architecture diagram created using Lucidchart. Original diagram available [here](https://lucid.app/invitations/accept/fd990180-b2a9-4089-81ce-3f4758d11c42)._

This architecture was set up with the following [architecture requirements](https://github.com/DigiPie/nodejs-mysql-cloudformation/issues/1) and [case study](https://gist.github.com/houdinisparks/b8dcd1d2b5b1179b45b0afe68351e027) in mind.

<details>
	<summary>VPC stack resources</summary>

- VPC
- Public Subnet 1 and 2: Resources inside are visible to the Internet
- Private Subnet 1 and 2: Resources can only be reached from other resources within the VPC
- Internet Gateway: Used by resources in the Public Subnet 1 and 2 for Internet-access
- NAT Gateway(s): Used as a proxy by resources in Private Subnet 1 and 2 (one NAT Gateway per AZ for Production)
- Bastion Security Group: Used by the Bastion stack
- Database Security Group: Used by the Database stack
- ELB and App Security Groups: Use by the Elastic Beanstalk stack

</details>

<details>
	<summary>Bastion stack resources</summary>

- Bastion Host: An EC2 instance in the Public Subnet 1
- Cloudwatch Alarms: For detecting invalid SSH attempts

</details>

<details>
	<summary>Database stack resources</summary>

- Master Database: Master RDS instance used for storing application data (Note: in a real-world scenario, use Amazon Aurora instead, which is sadly not in free-tier)
- Read Replica: Read Replica RDS instance used for heavy read operations on the database
- Standby Database: Standby RDS instance in the other Private Subnet/Availability Zone (AZ) for redudancy sake (only for Production)
- Database Subnet Group: For association with VPC Private Subnet 1 and 2

</details>

<details>
	<summary>Elastic Beanstalk stack resources</summary>

- Elastic Beanstalk Application
- Elastic Beanstalk Environment
- Auto-Scaling Group(s): Automatic scaling based on CPU utilization (2 ASGs in both AZ for Production)
- Elastic Load Balancer: Can be configured to be Network Load Balancer (Layer-4) or Application Load Balancer (Layer-7)

</details>

<details>
	<summary>Other services</summary>

- S3 Bucket: Used to store the initial app to be deployed to Elastic Beanstalk when the CloudFormation stack is built
- CodePipeline Pipeline: Used to continuously deploy the updated app from a GitHub repository to Elastic Beanstalk

</details>

## Getting started

Pre-requisites:

- [EC2 key pair](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-key-pairs.html)
- [npm](https://www.npmjs.com/get-npm)
- [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git/)
- [MySQL Workbench](https://www.mysql.com/products/workbench/) (optional)

### Setup VPC, Bastion and Database stacks

1. Clone this repository to your local disk.
1. Open the [AWS Cloudformation console](https://console.aws.amazon.com/cloudformation).
1. Click **Create stack** > **With new resources (standard)**.
1. Select **Prepare template**: 'Template is ready'.
1. Select **Template source**: 'Upload a template file'.
1. Launch the `vpc.cfn.yml` stack first and fill in the parameters accordingly. Keep track of what you fill as some of these values will be referenced later on.
1. After the VPC stack has a 'CREATE_COMPLETE' status, launch the `bastion.cfn.yml` and `db.cfn.yml` stacks. You can do so concurrently given the Bastion and Database stacks do not rely on each other; they only rely on the VPC stack.

### Setup Database table

1. Connect to the main RDS database via the bastion host with a database client such as [MySQL Workbench](https://www.mysql.com/products/workbench/). 
    - You cannot connect to the main and read replica databases directly given they are in the VPC's private subnets. You need to connect through the bastion host in the public subnet. 
    - See the _Connecting to Your Instances and Database_ section of [AWS: Building a VPC with the AWS Startup Kit](https://aws.amazon.com/blogs/startups/building-a-vpc-with-the-aws-startup-kit/).
    - You can find the required parameters for your database connection in the [AWS Cloudformation console](https://console.aws.amazon.com/cloudformation) > Select your Database stack > **Outputs**.
1. Execute the following SQL script to setup the database table used by the app-to-be-deployed:

    ```sql
    CREATE DATABASE test;

    USE test;

    CREATE TABLE users (
      id int(11) NOT NULL auto_increment,
      name varchar(100) NOT NULL,
      age int(3) NOT NULL,
      email varchar(100) NOT NULL,
      PRIMARY KEY (id)
    );
    ```

### Prepare your app for deployment

1. Clone the app repository [DigiPie/nodejs-mysql-aws-beanstalk](https://github.com/DigiPie/nodejs-mysql-aws-beanstalk) to your local disk.
1. In the app's root directory, zip the code by running the following npm command:

    ```shell
    npm run zip
    ```

1. Open the [AWS S3 Bucket console](https://s3.console.aws.amazon.com/s3/home).
1. Click **Create bucket**.
1. Click **Create bucket** using all default settings.
1. Upload the created zip file on the bucket.
1. Note down the bucket and zip file name (e.g. 'orca-dev' and 'nodejs-mysql-crud.zip' respectively). You will use them as parameters for your Elastic Beanstalk stack.

### Setup Elastic Beanstalk stack

1. Launch the `vpc.cfn.yml` stack first and fill in the parameters accordingly. Keep track of what you fill as some of these values will be referenced later on.
1. After the VPC stack has a 'CREATE_COMPLETE' status, launch the `bastion.cfn.yml` and `db.cfn.yml` stacks. You can do so concurrently given the Bastion and Database stacks do not rely on each other; they only rely on the VPC stack.

1. Return to the [AWS Cloudformation console](https://console.aws.amazon.com/cloudformation).
1. Launch the `elastic-beanstalk.cfn.yml` stack and fill in parameters as follows:

      - **AppS3Bucket**: Name of the S3 bucket that contains your zip file (e.g. 'orca-dev').
      - **AppS3Key**: Name of your zip file (e.g. 'nodejs-mysql-crud.zip').
      - **NetworkStackName**: Name of your VPC stack.
      - **DatabaseStackName**: Name of your DB stack.
      - **SSLCertificateArn**: Optional, can be empty.
1. Before launching your stack, select **Capabilities**: 'I acknowledge that AWS CloudFormation might create IAM resources.'.
1. After your Elastic Beanstalk stack has 'CREATE_COMPLETE' status, visit your application at `Output > EnvironmentURL`.

You're done!

## Continuous Deployment

To continuously deploy your application from GitHub to Elastic Beanstalk automatically on push to `master`, you can use either AWS CodePipeline or GitHub Action. In this case, we will be using AWS CodePipeline. For GitHub Action, refer to [DigiPie/nodejs-mysql-aws-beanstalk](https://github.com/DigiPie/nodejs-mysql-aws-beanstalk) for more instructions.

1. Open the [AWS CodePipeline console](http://console.aws.amazon.com/codepipeline).
1. Click **Create pipeline**.
1. Name your Pipeline and use the auto-filled Role name (e.g. 'AWSCodePipelineServiceRole-ap-southeast-1-test').
1. Select **Source provider**: 'GitHub (Version 1)'.
1. Click **Connect to GitHub**.
1. Select the **Repository** and **Branch**.
1. Use the default detection option: 'GitHub webhooks (recommended)'.
1. Click **Skip build stage**.
1. Select **Deploy provider**: 'AWS Elastic Beanstalk'.
1. Choose the appropriate **Region**, **Application name** and **Environment name**.
1. Click **Create pipeline**.
1. Test your pipeline by pushing a commit to `master`.

## References

- Sample application: [Node.js, Express & MySQL: Simple Add, Edit, Delete, View (CRUD)](http://blog.chapagain.com.np/node-js-express-mysql-simple-add-edit-delete-view-crud/)
- Sample AWS Elastic Beanstalk project: [amazon-archives/startup-kit-nodejs](https://github.com/amazon-archives/startup-kit-nodejs)
- Sample AWS CloudFormation templates: [aws-samples/startup-kit-templates](https://github.com/aws-samples/startup-kit-templates)
- Continuous Deployment pipeline: [AWS: Set up a Continuous Deployment Pipeline](https://aws.amazon.com/getting-started/hands-on/continuous-deployment-pipeline/)

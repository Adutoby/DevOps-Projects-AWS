# Project-4: Re-Factoring Web App on AWS Cloud[Cloud Native]

This strategy detail moving an application to the cloud and modifing its architecture to take full advantage of cloud-native features on AWS. Thereby improving Performance, agility and scalability while it reduces operational overhead and ensure business continuity.

## DevOps Project-4

Project Source: DevOps Project by [Imran Teli](https://www.udemy.com/course/devopsprojects/learn/lecture/23897820#overview)

## Pre-requisites

- AWS Beanstalk
- Amazon Simple Storage Service (S3)
- Amazon Relational Database System (RDS)
- Amazon Elastic Cache
- Amazon MQ
- Amazon Route53
- Amazon CloudFront
- Amazon Cloudwatch

### Other Tools

- jdk8
- maven

## Project Architecture

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/Project_Architectural_Diagram.png)

### Flow of Execution

- Login to AWS account
- Create key pair for beanstalk instance login
- Create a Security Group for Amazon ElastiCache, Amazon RDS and ActiveMQ
- Create RDS, Amazon Elastic Cache and Amazon Active MQ
- Create Elastic beanstalk Environment
- Update SG of Backend to allow traffic from beanstalk SG
- Update SG of Backend to allow internal traffic
- Launch Ec2 -Instance for DB Initialization
- Login to the instance and initialize RDS DB
- Change health check on beanstalk to /login
- Add 433 https Listener to ELB
- Build Artifact with Backend Information
- Deploy Artifact to Beanstalk
- Create a content delivery network (CDN) with SSL Certificate
- Update entry in Route 53
- Test the url

## Step 1: Create Key pair for Beanstalk EC2 Login

Create a key pair that will be used to log in to Elastic Beanstalk. Navigate to EC2 on AWS console, select key pair, and click create key pair.

    Name: vprofile-bean-key

Ensure to take a note of the location on your machine where the private key is downloaded as it will be needed to be able to  
login via SSH into the instance.

## Step 2: Create Security Group for ElastiCache, RDS and ActiveMQ (Backend Services)

Create a Security Group with the name vprofile-backend-SG to allow traffic on port 22 from its IP address.
Once created, edit inbound rules to allow all traffic from its own  security group that was just created.

## Step 3: Create Backend Services ( RDS Database)

Navigate to RDS on the AWS console,

### - Create Subnet Group:

Using  these properties, create a  Subnet Groups

    Name: vprofile-rds-sub-grp
    AZ: Select All
    Subnet: Select All

### - Create a Parameter Group

Create a parameter group that will be used with our RDS instance. we will be able to update the parameter of our RDS instance.

    Parameter group family: mysql5.7
    Type: DB Parameter Group
    Group Name: vprofile-rds-para-grp

### - Create Database

Create RDS instance with below properties:

    Method: Standard Create
    Engine Options: MySQL
    Engine version: 5.7.34
    Templates: Free-Tier
    DB Instance Identifier: vprofile-rds-mysql
    Master username: admin
    Password: Auto generate psw
    Instance Type: db.t2.micro
    Subnet grp: vprofile-rds-sub-grp
    SecGrp:  vprofile-backend-SG
    No public access
    DB Authentication: Password authentication
    Additional Configuration
    Initial DB Name: accounts
    Keep or add additional configuration according to your preference

Click the create database button. Click View credential details and Ensure to take note of the auto-generated db password.
This will be used in our application properties configuration file. (It will take some time for the RDS database to be created)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/RDSdatabase.png)

## Create ElasticCache

Search for Amazon ElasticCache in your AWS console, and proceed to

### - Create a Parameter Group

Creating parameter group for  ElastiCache instance

    Name: vprofile-memcached-para-grp
    Description: vprofile-memcached-para-grp
    Family: memcached1.4

### - Create Subnet Group

Creating  Subnet Groups with the below properties:

    Name: vprofile-memcached-sub-grp
    AZ: Select All
    Subnet: Select All

### - Create Memcached Cluster

On the dashboard, click Get Started, the drop down on Create cluster and then click the Memcached clusters and enter your details (note it will also take some time to create)
    Name: vprofile-elasticache-svc
    Engine version: 1.4.5
    Parameter Grp: vprofile-memcached-para-grp
    NodeType: cache.t2.micro
    number of Nodes: 1
    SecGrp: vprofile-backend-SG
    Tag: Name - vprofile-elasticache-svc

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/Memcached.png)

## Step 4: Create Amazon MQ

Search for Amazon MQ service on the AWS console, click on get started and create Rabbit MQ service Using below details

    Engine type: RabbitMQ
    Single-instance-broker
    Broker name: vprofile-rmq
    Instance type: mq.t3.micro
    username: rabbit
    psw: Texasboy1234
    Additional Settings:
    private Access
    VPC: use default
    SEcGrp: vprofile-backend-SG
    Tag: Name vprofile-rmq01

**`Ensure to keep a note of the username and password of your services.`**

## Step 5: DB Initialization

Copy the  RDS instance endpoint.

`vprofile-rds-mysql.cl4gecxfj3xt.us-east-1.rds.amazonaws.com`

Navigate to Instance and create an EC2 instance that will be used to initialize the DB, this instance will be terminated after our initialization.

    Name: mysql-client
    OS: ubuntu 18.04
    t2.micro
    SecGrp: mysql-client-SG
    Keypair: vprofile-prod-key

 Userdata:

    #! /bin/bash
    apt update -y
    apt install mysql-client -y

SSH into MySQL-client instance.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/sshintomysqlclient.png)

Check mysql version is installed with **`mysql -v`**
Update vprofile-backend-SG ie, edit Inbound rule to allow connection on port 3306 from mysql-client-SG we created for the instance.
Log in to the database, and connect with the below scripts. Ensure to use your RDS endpoint, username, and password credentials.

    mysql -h vprofile-rds-mysql.cl4gecxfj3xt.us-east-1.rds.amazonaws.com -u admin -p<db_password>
    mysql> show databases;

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/RDSlogincheck.png)

To initiate our database we need to clone our source code into the instance.

    git clone https://github.com/rumeysakdogan/vprofileproject-all.git
    cd vprofileproject-all
    git checkout aws-Refactor
    cd src/main/resources

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/clonerepo.png)

    mysql -h vprofile-rds-mysql.cl4gecxfj3xt.us-east-1.rds.amazonaws.com -u admin -pyOtlkK8l13npfKtbGgmg account < db_backup.sql
    mysql -h vprofile-rds-mysql.cl4gecxfj3xt.us-east-1.rds.amazonaws.com -u admin -pyOtlkK8l13npfKtbGgmg accounts
    show tables;

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/initializedb.png)

## Step 6: Create an Elastic Beanstalk Environment

With the backend services all ready, copy all of  their endpoints. This information will be used in our application.properties file.

    `RDS: vprofile-rds-mysql.cl4gecxfj3xt.us-east-1.rds.amazonaws.com:3306`
    `ActiveMQ: b-e2253522-105e-45f4-ba0b-c624eb588fac.mq.us-east-1.amazonaws.com:5671`
    `ElasticCache: vprofile-elasticache-svc.pxm6ij.cfg.use1.cache.amazonaws.com:11211`

### - Creating our  Application

AWS documentation details what Elastic Beanstalk is and you can read more [HERE](https://aws.amazon.com/elasticbeanstalk/)
Search for Amazon Elastic Beanstalk on your AWS console to create one.
We will choose Tomcat as the platform since we are running our application on Tomcat

    Name: vprofile-java-app
    Platform: Tomcat
    keep default for platform branch and platform version
    Application code: sample application
    Click Configure more options:
    select Custom configuration

    Edit Instances
    EC2 SecGrp: vprofile-backend-SG
    Keep others as default and save

    Edit Capacity
    LoadBalanced
    Min:2
    Max:4
    InstanceType: t2.micro
    metric: networkout
    Keep others as default and save

    Edit Rolling updates and deployments
    Deployment policy: Rolling
    Percentage :50 %
    Keep others as default and save

    Edit Security
    EC2 key pair: vprofile-bean-key

### - Update Backend SecGrp & ELB

For our application instances created by BeanStalk to communicate with Backend services, we need to update **`vprofile-backend-SG`** to allow connection from our app-SG created by Beanstalk on ports `3306`, `11211` and `5671`

  Custom TCP 3306 from Beanstalk SG (find the id from EC2 insatnces created by beanstalk)
  Custom TCP 11211 from Beanstalk SG
  Custom TCP 5671 from Beanstalk SG

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/ELB-SG-configuration.png)

Next, we need to configure the application environment load balancer.
To do this, navigate to the Elastic Beanstalk service in the AWS console, under the app environment, click Configuration and make changes to the Listener and processes section, and apply them.

  Add to Listener HTTPS port 443 with SSL cert
  Processes: Health check path : /login
  Stickiness policy: enabled

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/ELB-App-ssl-update.png)

## Step 7: Build and Deploy Artifact

Update the application.properties file with the correct endpoints and username and password.
This is found in the src/main/resources directory of the source code cloned to your local system.

    vim src/main/resources/application.properties

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/Editapp-properties.png)

CD back to the  root directory of the project where the **`pom.xml`** file is located.
Run **`mvn install`** command to build the artifact.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/mvnistall.png)

## - Upload Artifact to Elastic Beanstalk

Navigate back to the AWS console Elastic Beanstalk, click on the Application versions, and select the Upload button on the top right corner to upload the artifact from your local. It will automatically upload the artifact to an S3 bucket created by Elastic Beanstalk.

Select the uploaded application and click Deploy.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/Deplyartifact.png)

Verify that the application deployed successfully.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/Artifactdeployedsucess.png)

Click on the endpoint to see the login page.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/loginpage.png)

## Step 8: Create DNS Record in Route53 for the Application

Create a Cname record on route 53 pointing to the Elastic Beanstalk endpoint allowing us to reach the application securely with our DNS name
verify by opening a browser and entering the URL **`vprofile.(domainname)`**

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/Cname-record-route53.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/dnsverification.png)

Verify RDS

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/httpslongincheck.png)

Validate Active MQ service

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/ActiveMQvalidate.png)

Validate Database

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/dnsverification.png)

Validate ElasticCache

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/validateelasticcache.png)

Validate Vprofile Homepage

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/validatehomepage.png)

## Step 9: Create Cloudfront Distribution for CDN

Cloudfront is a Content Delivery Network service of AWS. It uses for global content distribution using Edge Locations around the world giving an excellent user experience with top-notch performance and very low latency.

Attach Cloudfront to Beanstalk
![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/Cloudfrontsetup.png)

Revalidate the application (URL) using another browser, and log in to check that all services are communicating and working as expected

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/cloudfrontvalidation.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-4:%20AWS-Refactoring/Images/httpslongincheck.png)

## Finally Step 10: Clean-up

Ensure to **delete all resources** created throughout the project to avoid charges.

Connect with me on [LinkedIn](https://www.linkedin.com/in/oluwatobiloba-adu/)
  
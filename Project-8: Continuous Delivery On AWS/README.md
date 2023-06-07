# Continuous Delivery on AWS Cloud[Java Application]

### DevOps Project-8
Project Source: [DevOps Project](https://www.udemy.com/course/devopsprojects/learn/lecture/33799976#overview) by [Imran Teli](https://www.udemy.com/course/devopsprojects/learn/lecture/23897820#overview)

## Architectural Diagram

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/ArchitecturalDiagram1.png)

### Prerequisite

   -  AWS Account
   
   -  AWS Cloud Services
   
   AWS CodeCommit ( as Version control System Repository)
    
   AWS Codebuild ( to Build services from AWS)
    
   AWS Codedeploy ( as Artifact Deployment Service)
    
   AWS Codeartifact ( Maven Repository for Dependencies)
    
   S3 Bucket ( deploy artifact for storage)
    
   AWS Codepipeline ( Service to integrate all jobs together)
    
  -  AWS CLI
  
  -  Git
  
  -  SolarCloud
  
  -  Checkstyle

## Flow of Execution
1. Login to AWS Account

2. Code Commit Setup

    a. Create Codecommit repo

    b. Sync it with local repository

3. Code Artifact Setup

    a. Create repository

    b. Update settings.xml filr in source code top level directory

    c. Update pom.xml file with repo details

    d. Generate token and store in SSM Parameter Store

4. Sonar Cloud Setup

    a. Create sonar cloud account

    b. Generate token and store in SSM Parameter store

    c. Create Build project

    d. Update codebuild role to access SSM parameter store

5. Create Notification on AWS SNS

6. Setup Build Project

    a. Update pom.xml with artifact version

    b. Create variables in SSM Parameter store

    c. Create build Project

    d. Update Codebuild role`to access SSM parameter store

7. Create Pipeline

    a. Codecommit

    b. Test code

    c. Build

    d. Deploy to s3 bucket

8. Test Pipeline.

9. Create Beanstalk and RDS.

10. Update RDS Sec grp.

11. Deploy DB in RDS.

    a. Switch to cd-aws branch.

12. Update settings.xml and pom.xml files.

13. Create another build job to create artifact with buildspec file

14. Create a deploy job to beanstalk.

15. Upload Screenshot to s3 Bucket.

16. Create a build job for software testing.

17. Update Pipeline.

18. Test Pipeline.

This project detail the comprehensive CICD process of Continuous integrating and delivery of a Java application. 
Here we will focus on **Step9 to Step18** as I have covered **Step1 - Step8** in my earlier project titled **[Continuous Integration on AWS](https://github.com/Adutoby/DevOps-Projects-AWS/tree/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud)**

## Step9: Create Elastic Beanstalk environment

Configure an environment using Sample application on your Elastic Beanstalk configuration environment.

        Name: vprofile-app
        Plateform: Tomcat
        Platform branch: Tomcat 8.5
        Capacity: LoadBalanced
            Min: 2
            Max: 4
        Keypair : Choose existing key-pair usedin previous steps

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/Beanstalkcreated.png)

## Step10: Create RDS MySQL Database

Create an RDS service using below information. Note: Remember to write down the Password. You will find it in the **[View Credential Details]**. 
This will be used later when setting up your database.

        Engine: MySQL
        version: 5.7
        Template: use the Free-Tier
        DB Identifier: vprofile-cicd-mysql
        credentials: admin
        Auto generate password (will take note of pwd once RDS is created)
        DB instance class: db.t2.micro
        Create new SecGrp Named: vprofile-cicd-rds-mysql-sg
        In Additional Configurations: initial db name "accounts"

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/viewcredentialdetails.png)

## Step11: Update RDS Security Group

Navigate to the Beanstalk instances created, and copy its Security group ID. 
Proceed to update the RDS Security Group Inbound rules to allow access for Beanstalk instances on port 3306.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/updateRDSsgwithbeanstalkinstance.png)

## Step12: Deploy DB in RDS.(DB initialization)

SSH into beanstalk to connect RDS instance. Install MySQL client in this instance to be able to connect RDS. 
Also install git to clone our source code and scripts to create schema in our database. Follow the commands below and ensure to update your credentials accordingly 
for RDS username, password and endpoint.

        sudo -i
        yum install mysql git -y
        mysql -h <RDS_endpoint> -u <RDS_username> -p<RDS_password>
        show databases;
        git clone https://github.com/Adutoby/vprofile-project-all.git
        cd vprofileproject-all/
        git checkout cd-aws
        cd src/main/resources
        mysql -h <RDS_endpoint> -u <RDS_username> -p<RDS_password> accounts < db_backup.sql
        mysql -h <RDS_endpoint> -u <RDS_username> -p<RDS_password>
        use accounts;
        show tables;

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/logintomysqlonbeanstalk.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/gitclone.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/DBinitializationaccountsetup1.png)

### Health check Update:

Return to the Elastic Beanstalk on your AWS console, in the your project environment Click on Configuration > edit the Instance traffic and scaling section, 
particularly Load balancer type > Click Processes, and update the Health check path with /login. Save and apply changes.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/Updatehealthcheck1.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/Updatehealthcheck2.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/Updatehealthcheck3.png)


## Step13: Update Code with pom & setting.xml

Navigate back to CodeCommit, and change to the cd-aws branch. Update the **pom.xml and settings.xml files** just like we did earlier in the project in the **`ci-aws branch`** and commit the changes.

In the **`pom.xml`**, add the correct url from your code artifact connection steps:

        <repository>
                <id>codeartifact</id>
                <name>codeartifact</name>
            <url>https://visualpath-<awsaccountid>.d.codeartifact.us-east-1.amazonaws.com/maven/maven-central-store/</url>
              </repository>

Also in the **`settings.xml`**, update url from code artifact.(Note, make sure to copy from the connection steps and update the url accordingly to your account)

        <?xml version="1.0" encoding="UTF-8"?>
        <settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">
          <servers>
                <server>
                    <id>codeartifact</id>
                    <username>aws</username>
                    <password>${env.CODEARTIFACT_AUTH_TOKEN}</password>
                </server>
            </servers>
        <profiles>
          <profile>
            <id>default</id>
            <repositories>
              <repository>
                <id>codeartifact</id>
            <url>https://visualpath-<awsaccountid>.d.codeartifact.us-east-1.amazonaws.com/maven/vprofile-maven-repo/</url>
              </repository>
            </repositories>
          </profile>
        </profiles>
        <activeProfiles>
                <activeProfile>default</activeProfile>
            </activeProfiles>
        <mirrors>
          <mirror>
            <id>codeartifact</id>
            <name>visualpath-vprofile-maven-repo</name>
            <url>https://visualpath-<awsaccountid>.d.codeartifact.us-east-1.amazonaws.com/maven/vprofile-maven-repo/</url>
            <mirrorOf>*</mirrorOf>
          </mirror>
        </mirrors>
        </settings>

## Step14: Build Job Setup

Navigate to Codebuild and change the source code for the earlier build jobs `Vprofile-Build` and `Vprofile-Build-Artifact` from the ci-aws to cd-aws since we intend to build from the **cd-aws branch**. 
This will trigger our our job from the new branch which is the **cd-aws branch**.

Click on build project > click on the Vprofile-Build project > click on edit and then source > change the source branch to cd-aws > click update changes. 
Repeat same for the Vprofile-Build-Artifact source branch.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/updatebuildprojectsorce1.png)

### Create BuildAndRelease Build Project

Create a new build project to deploy the artifacts to Elastic Beanstalk. used below details

        Name: Vprofile-BuildAndRelease
        Repo: CodeCommit
        Repository: vprofile-code-repo
        branch: cd-aws
        In the Environment
        Managed image: Ubuntu
        select Standard
        Image 5.0
        Use existing role from previous Build project which has access to SSM Parameter Store
        Buildspec: Insert build commands and switch to editor.
        (From source code copy the spec file under `aws-files/buildAndRelease_buildspec.yml`)
        LogGroup: vprofile-cicd-logs
        Streamname: BuildAndReleaseJob

Next, we need to create 3 new environment parameters which are used in the `BuilAndRelease_buildspec.yml` file in SSM Parameter store. See below. 
**If you experience error with artifact token, regenerate the codeartifact-token, edit and update it with the new token.**

        RDS-Endpoint: use String
        RDSUSER: Use String
        RDSPASS: use SecureString as this is the DB password.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/ParameterStoreupdated.png)

Time to Build our project.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/BuildAndReleasejobsuccessfull.png)

Our build project ran successfully! Good job to have made it to this point.
Create a build job for software testing.

Back in the Build Project, we will create another project that will run our Selenium automation scripts and store the artifacts in S3 bucket. 
First, we will need to create the S3 bucket.

        Name: vprofile-cicd-testoutput-rd (you need to have a unique name for your S3 bucket)
        Region: You bucket must be in same region where you create your pipeline

After the bucket is created, we will create a new Build project for Selenium Automation Tests using below details:

        Name: SoftwareTesting
        Repo: CodeCommit
        branch: seleniumAutoScripts
        in the Environment section:
        Operating systen: Windows Server 2019
        Runtime: Base
        Image: 1.0
        Use existing role from previous Build project which has access to SSM Parameter Store
        Insert build commands: 
        From source code we will get spec file under `aws-files/win_buildspec.yml`.
        Update url part to your Elastic Beanstalk URL.

        Artifacts:
        Type: S3
        Bucketname: vprofile-cicd-testoutput-toby
        Enable semantic versioning
        Artifcats packaging: zip
        LogGroup: vprofile-cicd-logs
        Streamname: 

## Step15: Create Pipeline

Create a CodePipeline with name of `vprofile-cicd-pipeline`. use below details for the other configurations

        Pipeline name: vprofile-cicd-pipeline
        Source provider: CodeCommit
        Repository name: vprofile-code-repo
        Branch name: cd-aws
        select Amazon CloudWatch Events
        select CodePipeline default

        Build
        Build Provider: CodeBuild
        Project Name: Vprofile-BuildAndRelease
        Build type: Single Build

        Deploy
        Deploy provider: Beanstalk
        Application name: vprofile-app
        Environment name: vprofile-app-env
        click on next and create pipeline

Stop the pipeline once it initiates and edit the pipeline to add more stages.

After Source stage, add the CodeAnalysis stage using below details

        Name: CodeAnalysis
        Action provider: CodeBuild
        Input artifacts: SourceArtifact
        Project name: Vprofile-Build
        Build type: single build

Add a second Stage also called CodeAnalysis stage to build and store the artifacts:

        Name: BuildAndStore
        Action provider: CodeBuild
        Input artifacts: SourceArtifact
        Project name: Vprofile-Build-artifact
        OutputArtifact: BuildArtifact

Then add the another after the BuildAndStore stage called the DeployTos3 Stage. This stage will deploy the store artifact to S3 bucket:

        Name: DeployToS3
        Action provider: Amazon S3
        Input artifacts: BuildArtifact
        Bucket name: vprofile98-build-artifact
        check the Extract file before deploy box

Edit output artifacts of Build and Deploy stages. Change Output artifact name as **`BuildArtifactToBeanStalk`**.

Again edit deploy stage, change the InputArtifact to **BuildArtifactToBeanStalk**.

Lastly, add after Deploy stage a stage to test the software. Use the details below.

        Name: Software Testing
        Action provider: CodeBuild
        Input artifacts: SourceArtifact
        ProjectName: SoftwareTesting

Save and Release change. The CodePipeline should start automatically.

## Step16: SNS Notification

Select our pipeline > Click Notify > Click Manage Notification. Create a new notification and save.

        vprofile-aws-cicd-pipeline-notification
        Select all
        Notification Topic: use same topic from CI pipeline

## Step17: Validate & Test

Pipeline was successful. All Stages ran as expected and were all successfully.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/pipelinebuildsuccessful.png)

Go back to your Elastic Beanstalk environment and grab the domain or endpoint, paste it in a browser and you should see your website up and running.

If you have multiple instances, you will need to enable the session stickiness. you will find this by editing the Instance traffic and scaling section in configuration, 
particularly Load balancer type > Click Processes > Action > edit, then find sessions,> then enable session stickiness by ticking the box. 
Ensure to do these steps if you have more than 1 instance set as minimum instance in your load balancer configurations.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/enablestickysession.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/loginpageok.png)

    Username: admin_vp
    password: admin_vp

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-8%3A%20Continuous%20Delivery%20On%20AWS/images/signinsuccessful.png)

**Congratulation!!!** making it to the end. 
I hope the work through was insightful and you hard fun completing the project. 
Lets connect on LinkedIn HERE.

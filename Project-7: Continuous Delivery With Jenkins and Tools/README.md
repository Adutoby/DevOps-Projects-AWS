# Continuous Delivery with Jenkins and Tools

### DevOps Project-7
Project Source: [DevOps Project](https://www.udemy.com/course/devopsprojects/learn/lecture/33799976#overview) by [Imran Teli](https://www.udemy.com/course/devopsprojects/learn/lecture/23897820#overview)

## Architectural Diagram

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/ArchitecturalDiagram1.png)

## Continuous Delivery Flow

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/ArchitectureflowDiagram.png)

### Prerequisite

- Jenkins
- Sonarqube
- Nexus Sonartype Repository
- Maven
- Docker
- Git
- Slack
- Amazon Elastic Container Registry (ECR)
- Amazon Elastic Container Service (ECS)
- AWS CLI

### Flow of Execution.

1. Create/Update Github Webhook with Jenkins IP (Assuming you have completed Continuous Integration stage. see [Project 5](https://github.com/Adutoby/DevOps-Projects-AWS/tree/master/Project-5%3AContinues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack)
    
2. Copy Dockerfile files to project repository.
3. Prepare Jenkinsfile for staging and Production in source code.
4. AWS Steps
5. 
    a. Setup IAM User and ECR Repository
    
5. Jenkins Steps

    a. Install plugins
    
    i. Amazon ECR
    
    ii. Docker, Docker build & publish
    
    iii. Pipeline: AWS steps
    
6. Install Docker engine and AWS CLI on Jenkins server.
7. Write Jenkins for Build and publish image to ECR.
8. ECS Setup

    a. Cluster
    
    b. Task definition
    
    c. Service
    
9. Write the Code to deploy Docker image to ECS.
10. Repeat the step for Prod ECS cluster.
11. Promoting Docker images for Production Environment.


## Step1: Create Branches and setup WebHook On Github with Jenkins IP


Note: This Project is a continuation of the my blog [Project 5-(Continuous Integration Using Jenkins, Nexus, SonarQube and Slack)](https://github.com/Adutoby/DevOps-Projects-AWS/tree/master/Project-5%3AContinues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack)
Therefore I will be assuming you have the 3 servers for Jenkins,SonarQube and Nexus servers up and running and the security groups updated accordingly.

To be able to have establish connection between Github and Jenkins, there is need to update your Jenkins public IP address on Gitthub Webhook which will 
establish build triggers whenever code changes occurs. To do this, copy your Jenkins Public IP, the navigate to the project repository on Github, Click on 
settings of your project repo, click again on Webhooks, then click on edit and the ensure to replace the Ip with your new Jenkins public address. 
Finally click on update Webhook at the bottom of the page. Check to be sure success sign.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/webhooksjenkinsipupdate.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/webhookconnectionestablished.png)

Next we will create a new branch off our project repo using our code editor or terminal, name the new branch **`cicd-Jenkins`**

        git checkout ci-jenkins
        git checkout -b cicd-jenkins
        
        
## Step2: Copy Dockerfile files to working project repository.


Download the **Dockerfile** from the repository provided below, switch to the **`docker branch`** of the repo and download the zip file of the repository for that branch.
Save the downloaded zip file to a location on your local system where you can easily extract its contents

        https://github.com/Adutoby/vprofile-project-all

Next, we copy the **Dockerfile** to your source code in the **`cicd-jenkins`** branch on your local repository.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/copydockerfiletosourcecode.png)

## Step3: Prepare Jenkinsfile for staging and Production in source code.

Create two folder in the project working directory and not the zipped file. Name the StagePipeline and ProdPipeline respectively

        mkdir StagePipeline ProdPipeline
        cp Jenkinsfile StagePipeline 
        cp Jenkinsfile ProdPipeline 
        rm Jenkinsfile

These Pipelines were created to be use in the staging and production level Pipeline stages of our project respectively.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/gitadDockerfileStageprod1.png)

Validate, add and push the changes to new branch cicd-jenkins from your local repository to Github. run this command

        cat .git/config
        git add . && git commit -m "dockerfile, stagepipeline and prodpipeline added" 
        git push origin cicd-jenkins

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/gitadDockerfileStageprod2.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/gitadDockerfileStageprod3.png)


## Step4: Setup IAM User and ECR


Create IAM user for Jenkins with Programmatic access keys and add **`ECRfullaccess and ECSfullaccess policies.`** follow these steps.

Navigate to IAM on your AWS console > click User >click Add User and give your user a name and click next > click add policies directly and search for 
these 2 policies and tick the boxes (AmazonECS_FullAccess and AmazonEC2ContainerRegistryFullAccess) and click next > click on create user.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/IAMuserforjenkinspolicy1.png)

Next, create a programmatic access key for the user **(Note Never search these access keys with anyone or posted to public spaces)** follow these steps

Click on the user you just created > click on secret credentials > click on Create Access key > click on command line interface > tick the caution box and click next. 
Your access key will be generated. Ensure to keep them securely and safely.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/IAMuserforjenkinspolicy2.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/IAMuserforjenkinspolicy3.png)

Create Private ECR repository to store our Docker images, name it **`vprofileappimg`** .

Navigate ECR on your AWS console and follow these steps. Click on get started > select a private repository option > give it a name of your choice **vprofileappimg** 
and click the create repository at the bottom of the page.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/CreateECR1.png)


## Step5: Jenkins Configurations


Paste your Jenkins <publicIP:8080> on a browser and login to your Jenkins. Then download these plugins below. following these steps

Click on manage Jenkins > Mange Plugins > Available Plugins > Search for each of the plugins and install without restart.

**Docker Pipeline**
**CloudBees Docker build and Publish**
**Amazon ECR**
**Pipeline: AWS Steps**

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/Jenkinspluginsinstall.png)

Next is to add AWS Credentials to Global Credentials of Jenkins, Click manage Jenkins > click Credentials > Under global, select add credentials > 
Choose AWS Credentials and provide Access key and Secret key ID.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/awscredssetup1.png)


## Step6: Install Docker engine and AWS CLI on Jenkins server.


ssh into your Jenkins server and install docker. Ensure to always use docker from the official websites as they change frequently and often.[](https://docs.docker.com/engine/install/ubuntu/)

Once docker is installed, add Jenkins to the docker user group and restart your Jenkins

        usermod -aG docker jenkins
        systemctl restart jenkins


## Step7: Docker Build Pipeline


Return to your project working directory and edit the Jenkinsfile in the StagePipeline. Add environmental variables to the Jenkinsfile. 
These variables are your registryCredential (this is the AWS credentials that was setup in Jenkins, ensure to use the format below **`ecr:region:credentialIDnameonjenkins`** 
and use the same name as the ID name that was saved in Jenkins). Second is the appRegistry url and third vprofileRegistry (this is the url of your container registry)

Observe to ensure they are correct in your Jenkinsfile, also remember to update all other variables accordingly expecially NEXUS parameters.

        def COLOR_MAP = [
            'SUCCESS': 'good', 
            'FAILURE': 'danger',
        ]
        pipeline {
            agent any
            tools {
                maven "MAVEN3"
                jdk "OracleJDK8"
            }

            environment {
                SNAP_REPO = 'vprofile-snapshot'
          NEXUS_USER = 'admin'
          NEXUS_PASS = 'adutoby'
          RELEASE_REPO = 'vprofile-release'
          CENTRAL_REPO = 'vpro-maven-central'
          NEXUSIP = '172.31.90.229'
          NEXUSPORT = '8081'
          NEXUS_GRP_REPO = 'vpro-maven-group'
                NEXUS_LOGIN = 'nexuslogin'
                SONARSERVER = 'sonarserver'
                SONARSCANNER = 'sonarscanner'
                registryCredential = 'ecr:us-east-1:awscreds'
                appRegistry = 'yourawsaccountID.dkr.ecr.us-west-1.amazonaws.com/vprofileappimg'
                vprofileRegistry = "https://yourawsaccountID.dkr.ecr.us-east-1.amazonaws.com"
            }

            stages {
                stage('Build'){
                    steps {
                        sh 'mvn -s settings.xml -DskipTests install'
                    }
                    post {
                        success {
                            echo "Now Archiving."
                            archiveArtifacts artifacts: '**/*.war'
                        }
                    }
                }

                stage('Test'){
                    steps {
                        sh 'mvn -s settings.xml test'
                    }

                }

                stage('Checkstyle Analysis'){
                    steps {
                        sh 'mvn -s settings.xml checkstyle:checkstyle'
                    }
                }

                stage('Sonar Analysis') {
                    environment {
                        scannerHome = tool "${SONARSCANNER}"
                    }
                    steps {
                       withSonarQubeEnv("${SONARSERVER}") {
                           sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                           -Dsonar.projectName=vprofile \
                           -Dsonar.projectVersion=1.0 \
                           -Dsonar.sources=src/ \
                           -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                           -Dsonar.junit.reportsPath=target/surefire-reports/ \
                           -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                           -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                      }
                    }
                }

                stage("Quality Gate") {
                    steps {
                        timeout(time: 1, unit: 'HOURS') {
                            // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                            // true = set pipeline to UNSTABLE, false = don't
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }

                stage("UploadArtifact"){
                    steps{
                        nexusArtifactUploader(
                          nexusVersion: 'nexus3',
                          protocol: 'http',
                          nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                          groupId: 'QA',
                          version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                          repository: "${RELEASE_REPO}",
                          credentialsId: "${NEXUS_LOGIN}",
                          artifacts: [
                            [artifactId: 'vproapp',
                             classifier: '',
                             file: 'target/vprofile-v2.war',
                             type: 'war']
                          ]
                        )
                    }
                }

                stage('Build App Image') {
                    steps {
                        script {
                            dockerImage = docker.build( appRegistry + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
                        }
                    }
                }

                stage('Upload App Image') {
                  steps{
                    script {
                      docker.withRegistry( vprofileRegistry, registryCredential ) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                      }
                    }
                  }
                }

            post {
                always {
                    echo 'Slack Notifications.'
                    slackSend channel: '#jenkins-cicd-tabnob',
                        color: COLOR_MAP[currentBuild.currentResult],
                        message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
                }
            }
        }

Commit/push the changes to our GitHub repository.

        Name: vprofile-cicd-pipeline-docker
        Type: Pipeline
        Build Trigger : GitHub hook trigger for GITScm polling
        Pipeline : pilpeline script from SCM
        URL : copy and paste the SSH url from GitHub for the cicd-jenkins branch
        crdentials: githubloginpass (the github credentials setup for jenkins access)
        branch: cicd-jenkins
        ScriptPath: StagePipeline/Jenkinsfile

Create a new pipeline in Jenkins with the below information

Click on **Build Now** on the just created **vprofile-cicd-pipeline-docker Pipeline.**
Observe to see the pipeline triggered a build job.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/StagePiplinebuildsuccessfull1.png)

Pipeline was successfully built and image uploaded to ECR repository.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/ECRimageuploadedsuccesfully.png)


## Step8: AWS ECS Setup


We now create ECS Cluster for our Staging environment. To do this, navigate to ECS on your AWS console. and follow these steps

Click on Cluster > Click on create cluster > select your VPC and Subnets > Select Fargate which follows Server-less model > and Click Create

        Cluster Name: vprostaging
        VPC : select your vpc

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/CreateECS1.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/createECS2.png)

Next is to create a **Task definition** that will our service will use to create and manage our app container. Use the inputs below

        Name: vproappstagetask
        containerName: vproapp
        Port: 8080
        Image URI: paste from ECR
        Environment: Fargate 1 vCPU, 2 GB memory
        click next to review and then click create

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/creattaskdefination1.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/createtaskdefination2.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/createtaskdefination3.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/createtaskdefination4.png)

Task Definition is now created

Next we create a **Service** now with these inputs .

        Name:  vproappstagesvc
        Replica: 1
        task definition: vproappstagetask
        LoadBalancer: create new
        target group vproappstagetg HTTP 80
        secGrp: vproappstagesg
        HTTP 80
        Health check: /login
        Grace period: 30

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/Createcervice1.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/Createcervice2.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/Createcervice3.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/Createcervice4.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/Createcervice5.png)

Once this is completed, we need to update the ports of both Target group and Security group to allow traffic from port 8080 which is our app port. 
To do that, navigate the EC2 on your console > Click on Target Group > select the Target Group created for your Service > Click on Health Check > Click on Edit > 
Click on Advanced Health Check Settings > Overide > Change the port to 8080 > and Save Changes.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/updatetargetgroup1.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/updatetargetgroup2.png)

To update port of the Security group to 8080 Click on Security Group of the vproappstagesg and Add Rule to allow traffic on port 8080 from anywhere on IPv4 and IPv6.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/updatesecuritygroupECS1.png)

Our Service is now Running, check app from a browser using either ALB url or the Public IP address:8080.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/taskdefinationcreated.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/apprunningpage2.png)


## Step9: Pipeline for ECS (Code to deploy Docker image to ECS.)

Next we will write and add a deploy stage to StagePipeline Jenkinsfile. This will allow for automatic deployment of our container to ECS. 
We will also add two new variables.

1. Add the newly created cluster name to our environmental variable code on our StagePipeline Jenkinsfile.

         cluster = "vprostaging"

2. Add the newly created service name also to our environmental variable code on our StagePipeline Jenkinsfile.

         service = "vproappstagesvc"

Here is the complete code

        def COLOR_MAP = [
            'SUCCESS': 'good', 
            'FAILURE': 'danger',
        ]
        pipeline {
            agent any
            tools {
                maven "MAVEN3"
                jdk "OracleJDK8"
            }

            environment {
                SNAP_REPO = 'vprofile-snapshot'
          NEXUS_USER = 'admin'
          NEXUS_PASS = 'adutoby'
          RELEASE_REPO = 'vprofile-release'
          CENTRAL_REPO = 'vpro-maven-central'
          NEXUSIP = '172.31.90.229'
          NEXUSPORT = '8081'
          NEXUS_GRP_REPO = 'vpro-maven-group'
                NEXUS_LOGIN = 'nexuslogin'
                SONARSERVER = 'sonarserver'
                SONARSCANNER = 'sonarscanner'
                registryCredential = 'ecr:us-east-1:awscreds'
                appRegistry = 'yourawsaccountID.dkr.ecr.us-east-1.amazonaws.com/vprofileappimg'
                vprofileRegistry = "https://yourawsaccountID.dkr.ecr.us-east-1.amazonaws.com"
                cluster = "vprostaging"
                service = "vproappprodsvc"
            }

            stages {
                stage('Build'){
                    steps {
                        sh 'mvn -s settings.xml -DskipTests install'
                    }
                    post {
                        success {
                            echo "Now Archiving."
                            archiveArtifacts artifacts: '**/*.war'
                        }
                    }
                }

                stage('Test'){
                    steps {
                        sh 'mvn -s settings.xml test'
                    }

                }

                stage('Checkstyle Analysis'){
                    steps {
                        sh 'mvn -s settings.xml checkstyle:checkstyle'
                    }
                }

                stage('Sonar Analysis') {
                    environment {
                        scannerHome = tool "${SONARSCANNER}"
                    }
                    steps {
                       withSonarQubeEnv("${SONARSERVER}") {
                           sh '''${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=vprofile \
                           -Dsonar.projectName=vprofile \
                           -Dsonar.projectVersion=1.0 \
                           -Dsonar.sources=src/ \
                           -Dsonar.java.binaries=target/test-classes/com/visualpathit/account/controllerTest/ \
                           -Dsonar.junit.reportsPath=target/surefire-reports/ \
                           -Dsonar.jacoco.reportsPath=target/jacoco.exec \
                           -Dsonar.java.checkstyle.reportPaths=target/checkstyle-result.xml'''
                      }
                    }
                }

                stage("Quality Gate") {
                    steps {
                        timeout(time: 1, unit: 'HOURS') {
                            // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                            // true = set pipeline to UNSTABLE, false = don't
                            waitForQualityGate abortPipeline: true
                        }
                    }
                }

                stage("UploadArtifact"){
                    steps{
                        nexusArtifactUploader(
                          nexusVersion: 'nexus3',
                          protocol: 'http',
                          nexusUrl: "${NEXUSIP}:${NEXUSPORT}",
                          groupId: 'QA',
                          version: "${env.BUILD_ID}-${env.BUILD_TIMESTAMP}",
                          repository: "${RELEASE_REPO}",
                          credentialsId: "${NEXUS_LOGIN}",
                          artifacts: [
                            [artifactId: 'vproapp',
                             classifier: '',
                             file: 'target/vprofile-v2.war',
                             type: 'war']
                          ]
                        )
                    }
                }

                stage('Build App Image') {
                    steps {
                        script {
                            dockerImage = docker.build( appRegistry + ":$BUILD_NUMBER", "./Docker-files/app/multistage/")
                        }
                    }
                }

                stage('Upload App Image') {
                  steps{
                    script {
                      docker.withRegistry( vprofileRegistry, registryCredential ) {
                        dockerImage.push("$BUILD_NUMBER")
                        dockerImage.push('latest')
                      }
                    }
                  }
                }

                stage('Deploy to ECS staging') {
                    steps {
                        withAWS(credentials: 'awscreds', region: 'us-east-1') {
                            sh 'aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment'
                        } 
                    }
                }
            }
            post {
                always {
                    echo 'Slack Notifications.'
                    slackSend channel: '#jenkins-cicd-tabnob',
                        color: COLOR_MAP[currentBuild.currentResult],
                        message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
                }
            }
        }

Commit/push the your changes to GitHub. This will trigger the pipeline job automatically.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/dockerimageECS.png)

Pipeline was built successfully and a notification was sent to Slack.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/ECSdeployslacknotification.png)


## Step10: Promote to Production


For the production Environment we will create a new ECS cluster entirely separate from the staging environment. 
After which we will create the Task definition.

        Name: vproprodtask
        containerName: vproapp
        Port: 8080
        Image URI: paste from ECR
        Environment: Fargate 1 vCPU, 2 GB memory

After the task definition, we will create a service for the prod env.

        Name:  vproappprodsvc
        Replica: 1
        task definition: vproappprodtask
        LoadBalancer: create new alb
        name: vproappprodalb
        target group vproappprodtg HTTP 80
        secGrp: vproappprodsg
        HTTP 80
        Health check: /login
        Grace period: 30

Just as we did for staging environment, we will need to update port to 8080 in both Target group and Security group of the Prod env.

We will then create a new branch from cicd-jenkins branch. Name the branch **Prod**.

        git checkout -b Prod

We will now need to create new Jenkinsfile in ProdPipeline directory with the code detailed below, commit the changes and push the branch to GitHub.

Here is the updated Jenkinsfile code for prod

        def COLOR_MAP = [
            'SUCCESS': 'good',
            'FAILURE': 'danger',
        ]

        pipeline {
            agent any

            environment {
                cluster = "vproprod"
                service = "vproappprodsvc"
            }

            stages {
                stage('Deploy to Prod ecs') {
                  steps {
                withAWS(credentials: 'awscreds', region: 'us-west-1') {
                  sh 'aws ecs update-service --cluster ${cluster} --service ${service} --force-new-deployment'
                }
              }
             }

            }
            post {
                always {
                    echo 'Slack Notifications.'
                    slackSend channel: '#jenkins-cicd-tabnob',
                        color: COLOR_MAP[currentBuild.currentResult],
                        message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info available at: ${env.BUILD_URL}"
                }
            }
        }

Create new pipeline production job from Jenkins UI.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/prodbuildsuccessfully.png)

We should have our application up and running from ECS.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-7%3A%20Continuous%20Delivery%20With%20Jenkins%20and%20Tools/images/prodappsvcrunning.png)

This bring us to the end of this very interesting and in demand Skill. A skill every DevOps engineer should be very familiar with. 
I enjoyed documenting it and hope to have your feedback on how to improve the continuous delivery process using Jenkins and tools.

Thank you again for reading. Please don’t forget to connect with me on LinkedIn [HERE](https://www.linkedin.com/in/oluwatobiloba-adu/).

See you on the next project, Keep learning and practicing.

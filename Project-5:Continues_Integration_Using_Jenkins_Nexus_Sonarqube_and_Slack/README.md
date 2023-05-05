# Continuous Integration Using Jenkins, Nexus, Sonarqube and Slack

### DevOps Project-5
Project Source: DevOps Project by [Imran Teli](https://www.udemy.com/course/devopsprojects/learn/lecture/23897820#overview)

### Pre-requisites

- AWS Account
- GitHub account
- Jenkins
- Nexus
- SonarQube
- Slack

## Project Architecture

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/ArchitecturalDiagram.png)

## Continuous Integration Flow

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/Continue_integration_Flow.png)

## Flow of Execution

1. Login to AWS Account.
2. Create Your Key Pair.
3. Create Security Group.
- Jenkins, Nexus and Sonarqube
4. Create EC2 Instances with User-data.
- Jenkins, Nexus and Sonarqube
5. Post Installation.
- Jenkins setup and Plugin
- Nexus Setup and repository setup
- Sonarqube login test
6. Git.
- Create a github repository and migration code
- integrate github repo with Vs-code and test it 
7. Build Job with Nexus integration.
8. Github Webhooks.
9. Sonarqube Server integration stage.
10. Nexus Artifact upload stage.
11. Slack Notification.
 
## Step1 and 2: Log into your AWS account and  Create Key Pair

Navigate to EC2, Key pair and Create a keypair. Download the private key to your local system. 
The Created key will be used to ssh into your servers once provisioned.

## Step3: Create Security Groups for Jenkins, Nexus and SonarQube

### Jenkins Security Group
Configure inbound rule to allow ssh login on port 22 and Allow Jenkins on port 8080 from anywhere. 
We will be updating the Jenkins security group rule to allow traffic on port 8080 via the sonerqube security group.

    Name: Jenkins-SG
    Allow: SSH from MyIP on port 22
    Allow: 8080 from Anywhere

### Nexus Security Group
Configure inbound rule to allow ssh login on port 22, also configure port 8081 to be accessible from the browser and
port 8081 from the the configured Jenkins-SG.

     Name: Nexus-SG
     Allow: SSH on port 22 from MyIP
     Allow: 8081 from MyIP and Jenkins-SG
     
### SonarQube Security Group
Configure the inbound rule allow SSH on port 22, allow access from My IP on port 80 and  also from the configure Jenkins-SG

    Name: SonarQube-SG
    Allow: SSH from MyIP
    Allow: 80 from MyIP and Jenkins-SG
    
Once SonarQube-SG is created edit Jenkins-SG inbound rule to allow traffic from sonarqube security group. 
This allows for sonarqube send results back to our Jenkins server.

## Step4: Create EC2 instances 

We will create 3 instances for Jenkins, Nexus and SonarQube using user-data scripts 

### Jenkins Server Setup
Create Jenkins- Server 

      Name: jenkins-server
      AMI: Ubuntu 20.04
      Security Group: jenkins-SG
      InstanceType: t2.small
      KeyPair: awskeypair
      Additional Details: copy and paste below jenkins-setup script
      
**Jenkins Userdata script**

      #!/bin/bash
      sudo apt update -y
      sudo apt install openjdk-11-jre -y
      sudo apt install maven -y
      curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | sudo tee \
        /usr/share/keyrings/jenkins-keyring.asc > /dev/null
      echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
        https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
        /etc/apt/sources.list.d/jenkins.list > /dev/null
      sudo apt-get update
      sudo apt-get install jenkins
      sudo systemctl enable jenkins
      sudo systemctl start jenkins
      ###

### Nexus Server and repository Setup:
Create Nexus-server lunching an Amazon Linux-2 instance on AWS

      Name: nexus-server
      AMI: Amazon Linux-2
      InstanceType: t2.medium
      SecGrp: nexus-SG
      KeyPair: awskeypair (ensure to use your created keypair here)
      Additional Details: copy and paste below Nexus-setup script
      
**Nexus Userdata script**

      #!/bin/bash
      sudo yum install java-1.8.0-openjdk.x86_64 wget -y   
      mkdir -p /opt/nexus/   
      mkdir -p /tmp/nexus/                           
      cd /tmp/nexus/
      NEXUSURL="https://download.sonatype.com/nexus/3/latest-unix.tar.gz"
      wget $NEXUSURL -O nexus.tar.gz
      EXTOUT=`tar xzvf nexus.tar.gz`
      NEXUSDIR=`echo $EXTOUT | cut -d '/' -f1`
      rm -rf /tmp/nexus/nexus.tar.gz
      rsync -avzh /tmp/nexus/ /opt/nexus/
      sudo useradd nexus
      sudo chown -R nexus.nexus /opt/nexus 
      cat <<EOT>> /etc/systemd/system/nexus.service
      [Unit]                                                                          
      Description=nexus service                                                       
      After=network.target                                                            

      [Service]                                                                       
      Type=forking                                                                    
      LimitNOFILE=65536                                                               
      ExecStart=/opt/nexus/$NEXUSDIR/bin/nexus start                                  
      ExecStop=/opt/nexus/$NEXUSDIR/bin/nexus stop                                    
      User=nexus                                                                      
      Restart=on-abort                                                                

      [Install]                                                                       
      WantedBy=multi-user.target                                                      
      EOT
      echo 'run_as_user="nexus"' > /opt/nexus/$NEXUSDIR/bin/nexus.rc
      sudo systemctl daemon-reload
      sudo systemctl start nexus
      sudo systemctl enable nexus     

### Sonarqube server setup:
Create sonarqube-server lunched with an ubuntu 18.04 AMI 

      Name: Sonarqube-server
      AMI: Ubuntu 18.04
      InstanceType: t2.medium
      SecGrp: SonarQube-SG
      KeyPair: awskeypair (ensure to use your created keypair here)
      Additional Details: copy and paste below Sonarqube-setup script
      
**Sonarqube Userdata script**

      #!/bin/bash
      sudo cp /etc/sysctl.conf /root/sysctl.conf_backup
      cat <<EOT> /etc/sysctl.conf
      vm.max_map_count=262144
      fs.file-max=65536
      ulimit -n 65536
      ulimit -u 4096
      EOT
      sudo cp /etc/security/limits.conf /root/sec_limit.conf_backup
      cat <<EOT> /etc/security/limits.conf
      sonarqube   -   nofile   65536
      sonarqube   -   nproc    409
      EOT
      sudo apt-get update -y
      sudo apt-get install openjdk-11-jre -y
      sudo update-alternatives --config java
      #java -version
      sudo apt update -y
      wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
      sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
      sudo apt install postgresql postgresql-contrib -y
      #sudo -u postgres psql -c "SELECT version();"
      sudo systemctl enable postgresql.service
      sudo systemctl start  postgresql.service
      sudo echo "postgres:admin123" | chpasswd
      runuser -l postgres -c "createuser sonar"
      sudo -i -u postgres psql -c "ALTER USER sonar WITH ENCRYPTED PASSWORD 'admin123';"
      sudo -i -u postgres psql -c "CREATE DATABASE sonarqube OWNER sonar;"
      sudo -i -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE sonarqube to sonar;"
      systemctl restart  postgresql
      #systemctl status -l   postgresql
      netstat -tulpena | grep postgres
      sudo mkdir -p /sonarqube/
      cd /sonarqube/
      sudo curl -O https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.3.0.34182.zip
      sudo apt-get install zip -y
      sudo unzip -o sonarqube-8.3.0.34182.zip -d /opt/
      sudo mv /opt/sonarqube-8.3.0.34182/ /opt/sonarqube
      sudo groupadd sonar
      sudo useradd -c "SonarQube - User" -d /opt/sonarqube/ -g sonar sonar
      sudo chown sonar:sonar /opt/sonarqube/ -R
      sudo cp /opt/sonarqube/conf/sonar.properties /root/sonar.properties_backup
      cat <<EOT> /opt/sonarqube/conf/sonar.properties
      sonar.jdbc.username=sonar
      sonar.jdbc.password=admin123
      sonar.jdbc.url=jdbc:postgresql://localhost/sonarqube
      sonar.web.host=0.0.0.0
      sonar.web.port=9000
      sonar.web.javaAdditionalOpts=-server
      sonar.search.javaOpts=-Xmx512m -Xms512m -XX:+HeapDumpOnOutOfMemoryError
      sonar.log.level=INFO
      sonar.path.logs=logs
      EOT
      cat <<EOT> /etc/systemd/system/sonarqube.service
      [Unit]
      Description=SonarQube service
      After=syslog.target network.target
      [Service]
      Type=forking
      ExecStart=/opt/sonarqube/bin/linux-x86-64/sonar.sh start
      ExecStop=/opt/sonarqube/bin/linux-x86-64/sonar.sh stop
      User=sonar
      Group=sonar
      Restart=always
      LimitNOFILE=65536
      LimitNPROC=4096
      [Install]
      WantedBy=multi-user.target
      EOT
      sudo systemctl daemon-reload
      sudo systemctl enable sonarqube.service
      #systemctl start sonarqube.service
      #systemctl status -l sonarqube.service
      sudo apt-get install nginx -y
      sudo rm -rf /etc/nginx/sites-enabled/default
      sudo rm -rf /etc/nginx/sites-available/default
      cat <<EOT> /etc/nginx/sites-available/sonarqube
      server{
          listen      80;
          server_name sonarqube.groophy.in;
          access_log  /var/log/nginx/sonar.access.log;
          error_log   /var/log/nginx/sonar.error.log;
          proxy_buffers 16 64k;
          proxy_buffer_size 128k;
          location / {
              proxy_pass  http://127.0.0.1:9000;
              proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;
              proxy_redirect off;

              proxy_set_header    Host            \$host;
              proxy_set_header    X-Real-IP       \$remote_addr;
              proxy_set_header    X-Forwarded-For \$proxy_add_x_forwarded_for;
              proxy_set_header    X-Forwarded-Proto http;
          }
      }
      EOT
      sudo ln -s /etc/nginx/sites-available/sonarqube /etc/nginx/sites-enabled/sonarqube
      sudo systemctl enable nginx.service
      #systemctl restart nginx.service
      sudo ufw allow 80,9000,9001/tcp
      echo "System reboot in 30 sec"
      sleep 30
      reboot
      
## Step5: Post Installation Steps

### Jenkins Server setup and plugins :
SSH into the Jenkins server and validate the status of Jenkins with the below command. Status should be active (running). 

      sudo -i
      systemctl status jenkins      

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/sshjenkinsandjenkinsstatus.png)

Open a browser and enter the your jenkins-server IP address on port 8080 To get the initial Admin password from directory use the below command

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/Jenkinsloginpage.png)

Install recommended plugins and more importantly the below listed plugins as we will be using them on this project

**Maven Integration**

**Github Integration**

**Nexus Artifact Uploader**

**SonarQube Scanner**

**Slack Notification**

**Build Timestamp**


### Nexus Server and repository setup:
SSH into nexus server, check system status for nexus.

      sudo -i
      systemctl status nexus
      
Status should be active(running) as displayed below.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/sshnexusandnexusstatus.png)

Open your browser, input your nexus-server public IP address on port 8081 . To sign in you need to use this command  **cat `opt/nexus/sonatype-work/nexus3/admin.password`** to access the initial password . Follow the wizard to update your new password and ensure to disable anonymous access.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/Nexussigninpage.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/disableanonacesstobexus.png)

Select gear symbol and create repository. We will be using the repository to store our release artifacts.

      maven2 hosted
      Name: vprofile-release
      Version policy: Release

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/nexusrepositorygear.png)

Create another repository for maven2 proxy. This will store any dependencies required for our project. 
when the dependency needed, the proxy repository will checked in Nexus and downloaded. Proxy repository will download the dependencies
from maven2 central repository URL provided

      maven2 proxy
      Name: vpro-maven-central
      remote storage: https://repo1.maven.org/maven2/
      
Create another repository to store snapshot artifacts. All artifact with snapshot extension will be stored in this repository. 
Ensure to change the version policy to snapshot.

      maven2 hosted
      Name: vprofile-snapshot
      Version policy: Snapshot      
      
Lastly create a repository with the below configuration to group all your created repository together.

      Repository type: maven2 group
      Name: vpro-maven-group
      Member repositories: 
       - vpro-maven-central
       - vprofile-release
       - vprofile-snapshot      
       
![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/nexusrepovalidated.png)

### SonarQube Server login test:

ssh into your sonarqube server and validate sonar service status. Status should be active(running).

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/sshsonarqubeandstatus.png)

Open a new browser enter the public IP address of your sonarqube-server instance. 
Use username admin and password admin to login. Ensure to change your password. This is best the practice as default password aren't save.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/sonarqubelogintest.png)

## Step6: Create a repository in Github

Create a private repository in the Github for the project, then clone content from this project repo shown below:

      git clone -b ci-jenkins git@github.com:Adutoby/vprofile-project-all.git

## Step7: Build Job with Nexus Integration

We will require some dependencies to build our job on Jenkins server. This include JDK8 and Maven.

Navigate tp Manage Jenkins in your Jenkins UI, then to Global Tool Configuration. Under **JDK**, select **`Add JDK`** > Name it, 
uncheck install Automatically and under **JAVA HOME** provide the path for JDK-8.(this is discussed below)

We also need jdk8 installed and to do that we will need to ssh into our Jenkins server (instance) and run the following commands 


      sudo apt update -y
      sudo apt install openjdk-8-jdk -y
      
Since our application is using JDK8, we need to install Java8 on Jenkins. Follow these steps: Manage Jenkins > Global Tool Configuration select Install JDK8 manually, and specify its PATH in there.

Paste **`/usr/lib/jvm/java-1.8.0-openjdk-amd64`** in the path section to install jdk8

Enter **MAVEN3** and leave rest as they are and  Save the configuration

We also need to add Nexus credentials to be able  upload our artifacts, 
Therefor we will to add Nexus login credentials to Jenkins by navigating to **Manage Jenkins > Manage credentials > Global > Add Credentials used below information for the configuration.**

      username: admin
      password: enter the password you setup for nexus
      ID: nexuslogin
      description: nexuslogin

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/nexusintegrationtojenkins.png)

Next is to create a Jenkinsfile to build our pipeline. The variables mentioned in the `pom.xml` repository and `settings.xml` will be declared in Jenkinsfile and their values to be used during execution. 

Write the Pipeline code, save, commit and push to GitHub.

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
          }

          stages {
              stage('Build'){
                  steps {
                      sh 'mvn -s settings.xml -DskipTests install'
                  }
              }
          }
      }

Moving forward, we will then create a New Job in Jenkins with the below property and save them.
  
      Pipeline from SCM 
      Git
      URL: <url_from_project> I will use SSH link
      Crdentials: we will create github login credentials
      #### add Jenkins credentials for github ####
      Kind: SSH Username with private key
      ID: Gitpass
      Description: Gitpass
      Username: git
      Private key file: paste your private key here
      #####
      Branch: */ci-jenkins
      path: Jenkinsfile
      
Login Jenkins server via ssh and complete host-key checking stepup with the command below. The host-key will be stored in **`.ssh/known_host`** file. 
Only then will error be fixed.

      sudo -i
      sudo su - jenkins
      git ls-remote -h git@github.com:Adutoby/vprociproject.git HEAD
      
![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/githubidentitysavedtojenkins.png)

Build the pipeline see that ran and built successfully.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/piplinebuildsuccessful.png)

## Step8: GitHub Webhook 

We will automate the process of manual build done on the Jenkins UI by using Github webhook so that whenever a developer makes changes or/and 
commit to the repository, a pipeline job is triggered automatically.

To setup the webhooks, Copy your Jenkins URL, then  navigate to your Github repository settings. Go to the webhooks and follow the process below.
**Click setting > Webhooks > Add your <JenkinsURl>/github-webhook/ at the end of the JenkinsURL.**

Go back to Jenkins UI, to configure your pipeline job to set the build trigger to Build Trigger: GitHub hook trigger for GITScm polling and save the configuration

Next we will update the Jenkinsfile by adding a post action to our pipeline script and commit/push changes to GitHub. Once this is done an automatic build should be triggered if the configuration is okay.

      pipeline {
          agent any
          tools {
              maven "MAVEN3"
              jdk "OracleJDK8"
          }

          environment {
              SNAP_REPO = 'vprofile-snapshot'
        NEXUS_USER = 'admin'
        NEXUS_PASS = 'admin123'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUSIP = '172.31.5.4'
        NEXUSPORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
              NEXUS_LOGIN = 'nexuslogin'
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
          }
      }

Build job was triggered successfully and automatically

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/webhookjobtriggeredsuccess.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/weebhookjobbuiltsuccess.png)

## Step9: SonarQube Sever Integration and code analysis

Recall we added a unit job in the previous pipeline. This  Unit test and code coverage reports generated are located  under Jenkins workspace target directory. The reports are not human readable and so we will be needing a tool to scan and analyze the code to a human readable format the that will be done by our SonarQube server and a toll called sonarScanner. 

**To set this up:**
Navigate to Manage Jenkins > Click Global Tool Configuration and add the sonar scanner.

      Add: SonarQube scanner
      Name: sonarscanner
      and tick install automatically option
      
Next go to Configure system under the manage Jenkins and find sonarqube sever section and follow below configuration. Remember to save.

      Check mark the environment variables box
      Add sonarqube
      Name: sonarserver
      Server URL: http://<private_ip_of_sonar_server>
      Server authentication token: create the token from sonar website
      
Next we add sonar token to global credentials section. To do this navigate to the sonarqube UI, click on the admin name > my account > security > give your token a name and generate the token. copy the token and continue your global credentials configuration on Jenkins where you will follow below. Note that you will have to save without the secret and return to input secret to complete the configuration for it to work. Ensure to save.

      Kind: secret text
      Secret: paste the token generated 
      name: sonartoken
      description: sonartoken
      
To add sonarQube code for our pipeline and commit/push changes to GitHub.

      pipeline {
          agent any
          tools {
              maven "MAVEN3"
              jdk "OracleJDK8"
          }

          environment {
              SNAP_REPO = 'vprofile-snapshot'
        NEXUS_USER = 'admin'
        NEXUS_PASS = 'admin123'
        RELEASE_REPO = 'vprofile-release'
        CENTRAL_REPO = 'vpro-maven-central'
        NEXUSIP = '172.31.5.4'
        NEXUSPORT = '8081'
        NEXUS_GRP_REPO = 'vpro-maven-group'
              NEXUS_LOGIN = 'nexuslogin'
              SONARSERVER = 'sonarserver'
              SONARSCANNER = 'sonarscanner'
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
          }
      }
      
Observer to see our job was triggered successfully.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/sonarcodeupdatejobtriggersuccess.png)

Job was also built successfully

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/sonarcodejobbuiltsuccess.png)

On the SonarQube UI we should now see our project with the quality gate results Passed.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/Sonarqubequalityresult.png)

To create our own Quality Gates and add it to our project, we will need  create a Webhook in SonarQube to send the analysis results to Jenkins. 

To do this. Navigate on the sonarqube UI to **Quality gate > click on create > give your Quality gate a name `"vprofileQG"` and save.**
The click on add condition to give your rules. Select on over all codes set your rule and add condition.

Then attached the quality gate to your project. Go back to your **project > project settings > select quality gate > and select the quality gate you just created.**

Now setup the webhooks by navigating to project settings > select webhooks > select create > give a name **`jenkinswebhook`** use below sample url for in the URL section and create.

      http://<private_ip_of_jenkins>:8080/sonarqube-webhook

Update the Jenkinsfile by adding a new stage named "Quality Gate" to the pipeline and commit changes to GitHub.

      stage("Quality Gate") {
                  steps {
                      timeout(time: 1, unit: 'HOURS') {
                          // Parameter indicates whether to set pipeline to UNSTABLE if Quality Gate fails
                          // true = set pipeline to UNSTABLE, false = don't
                          waitForQualityGate abortPipeline: true
                      }
                  }
              }

Run the job. 
Build was successful!!

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/Qualitygatebuildsuccessful.png)

## Step10: Publish Artifact to Nexus Repo

Here we will be  automating the process of publishing the  latest artifact to Nexus repository after a successful build and to do this we will need to add Build-Timestamp to the  artifact name to get unique artifact each time a job is successfully built.

Click Manage Jenkins > Configure System under Build Timestamp, update the pattern as we desire.

**`yy-MM-dd_HHmm`**

Now we will add another stage to our pipeline called UPLOAD ARTIFACT save our changes, commit the changes and see the result.
 
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

Job was again built successfully!

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/Nexusartifactjobbuiltsuccess.png)

Our artifact was uploaded to Nexus repository.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/nexusartifactreleasevalidation.png)

## Step11: Slack Notification

Login to slack and create a workspace. We will then create a channel called **`jenkins-cicd-tabnob`** in our workspace.

Next we will add Jenkins app to slack. The step to do this available Searching  on the web so lets Google Slack app and then search for Jenkins, select the **Jenkins CI**. Choose the channel **`jenkins-cicd-tabnob`** you just created. It will give a setup instructions, then copy the Integration token credential ID . Ensure to save the settings before you leave the page.

Return to your Jenkins dashboard, Click Manage Jenkins > System configuration > Slack and Use the guides below.

      Workspace:  ( the workspace you created )
      credential: give a credential name
      default channel: #jenkins-cicd-tabnob
      
Next is to add our Slack token to Jenkins global credentials. Again you the guide 

      Kind: secret text
      Secret: the token copied should be pasted here.
      name: slacksecret
      description: slacksecret
![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/slackjenkinpluginconfirmation.png)   
      
Now update our Jenkinsfile script with a post installation step, commit and push the changes to GitHub which will automatically trigger 
a build job and we should receive the notification on slack accordingly.

      post{
              always {
                  echo 'Slack Notifications'
                  slackSend channel: '#jenkins-cicd-tabnob',
                      color: COLOR_MAP[currentBuild.currentResult],
                      message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
              }
          }
          
Notification was sent and received on slack. Great Job making it through the lab.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-5:Continues_Integration_Using_Jenkins_Nexus_Sonarqube_and_Slack/Images/slacknotificationmessagesuccess.png)

### Finally Clean-up

Ensure to delete all resources created throughout the project.

Thank you for reading. please react and send in your comments and don't forget to connect with me on [LinkedIn](https://www.linkedin.com/in/oluwatobiloba-adu/).

**Keep learning and practicing.**

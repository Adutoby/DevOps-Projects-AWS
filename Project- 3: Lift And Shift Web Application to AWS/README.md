# Lift and Shift Web Application Workload to the Cloud. Amazon Web Service (AWS)
## DevOps Project-3
#### Project Source: DevOps Project by Imran Teli

In this Project, We will be using the LIFT AND SHIFT strategy  for our web application setup. 
Lift And Shift is a strategy involves the migration of an application or a code base from a local setup(such as  data center or VMs) to a cloud 
environment,in this case AWS . I will be migrating a java based  application that was setup in my previous. Link to the project 1
This Project Aims to solve cost and scalability challenges building running your application locally (Data center). 
The cloud platform offers lots of benefits. Read more on Cloud Computing Setup and Benefits at a glance 

# AWS SERVICES USED

- Elastic Compute Cloud (EC2) 
- Elastic Load Balancer (ELB)
- Auto-scaling
- S3 bucket
- Amazon Certificate Manager (ACM)
- Route 53
- Security Group
- EFS

### Other Tools Used are
- DNS resolution
- jdk8
- maven

# Project Architecture on AWS

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/project%20architecture.png)

picture credit: rumeysakdogan

# Flow of Execution
1. Log into your AWS account
2. Create security groups for the backend services
3. Create Key pairs to login into server
4. Lunch EC2 instance with user data (BASH SCRIPTS)
5. Update IP to name mapping in route 53
6. Build Our Application from source code
7. Upload our Application to S3 bucket
8. Download artifact to Tomcat EC2 Instance
9. Setup ELB with HTTPS [certificate from (ACM)Amazon certificate manager
10. Map ELB Endpoint to website name in Amazon route 53 and Verify
11. Configure Auto Scaling Group for the Application Instance and Verify

# Step 1

Log in to your AWS account
Create a certificate on Amazon Certificate Manager and validate it with the Domain Name created on AWS Route53 for a secure connection (HTTPS)

# Step 2

Navigate to the EC2 section from the console page and create a security group for our load balancer where we allow the inbound rules for HTTP (port 80)
and HTTPS (port 443) on IPV4 and IPV6

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/vprofile-ELB-SG.png)

Create a security for our Tomcat instances and allow inbound rule on port 8080 from load balancer security group

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/tomcatinstancesSG.png)

Create security group for our back-end services rabbitMQ, Memcached and MySQL and add inbound rule for MySQL (port 3306), memcached (port 11211), 
RabbitMQ (port 5672) and allow from only tomcat security group. We will also allow all traffic from its security group after we have created the 
aforementioned security rules for the back-end services. To be able to log into the server port 22 should be configured from the instance IP 
addresses of the back-end instances once provisioned.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/backendservicesSG.png)

# Step 3

Create a key pair to login to the EC2 instances.
Launch an instance for DB instance on Centos 7 using the bash script in mysql.sh as the user-data to provision the service. 
Also add Inbound rule to vprofile-backend-SG for SSH on port 22 from the instance IP to be able to connect our db instance via SSH.

    Name: vprofile-db01
    Project: vprofile
    AMI: Centos 7
    InstanceType: t2.micro
    SecGrp: vprofile-backend-SG

In the advance detail section, scroll down to user data and past the bash script contained in mysql.sh. this will bootstrap the instance 
and provision mariadb when the server is up. Note it might take few minutes to be ready.

    #!/bin/bash
    DATABASE_PASS='admin123'
    sudo yum update -y
    sudo yum install epel-release -y
    sudo yum install git zip unzip -y
    sudo yum install mariadb-server -y

    # Starting & enabling mariadb-server
    sudo systemctl start mariadb
    sudo systemctl enable mariadb
    cd /tmp/
    git clone -b vp-rem https://github.com/devopshydclub/vprofile-repo.git
    #restore the dump file for the application
    sudo mysqladmin -u root password "$DATABASE_PASS"
    sudo mysql -u root -p"$DATABASE_PASS" -e "UPDATE mysql.user SET Password=PASSWORD('$DATABASE_PASS') WHERE User='root'"
    sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')"
    sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User=''"
    sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%'"
    sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
    sudo mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
    sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123'"
    sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123'"
    sudo mysql -u root -p"$DATABASE_PASS" accounts < /tmp/vprofile-repo/src/main/resources/db_backup.sql
    sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"

    # Restart mariadb-server
    sudo systemctl restart mariadb

    #starting the firewall and allowing the mariadb to access from port no. 3306
    sudo systemctl start firewalld
    sudo systemctl enable firewalld
    sudo firewall-cmd --get-active-zones
    sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
    sudo firewall-cmd --reload
    sudo systemctl restart mariadb

Once our instance is ready, SSH into the server and check if the script executed properly and the mysql status. 
follow the commands, change to root user, abd use the curl to check the user-data script and the sytemctl status mariadb to check 
the running state of the database.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/db01login.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/db01statusvalidation.png)

    ssh -i awskeypair.pem centos@<public_ip_of_instance>
    sudo -i
    curl http://169.254.169.254/latest/user-data
    systemctl status mariadb

Next, launch another Centos 7 EC2 instance for memchache using the memcache user-data for provisioning, use the same key pair, and security group used.

    Name: vprofile-mc01
    Project: vprofile
    AMI: Centos 7
    InstanceType: t2.micro
    SecGrp: vprofile-backend-SG
    UserData: memcache.sh

## User data for memcache
    #!/bin/bash
    sudo yum install epel-release -y
    sudo yum install memcached -y
    sudo systemctl start memcached
    sudo systemctl enable memcached
    sudo systemctl status memcached
    sudo memcached -p 11211 -U 11111 -u memcached -d

SSH into the server and check if user data script was executed properly and check status of memcache service if it is listening on port 11211.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/memcacheloginstatusvalidation.png)

Create another EC@ instance for RabbitMQ using rabbitmq.sh script as user data

    Name: vprofile-rmq01
    Project: vprofile
    AMI: Centos 7
    InstanceType: t2.micro
    SecGrp: vprofile-backend-SG
    UserData: rabbitmq.sh

## User data for RabbitMQ

    #!/bin/bash
    sudo yum install epel-release -y
    sudo yum update -y
    sudo yum install wget -y
    cd /tmp/
    wget http://packages.erlang-solutions.com/erlang-solutions-2.0-1.noarch.rpm
    sudo rpm -Uvh erlang-solutions-2.0-1.noarch.rpm
    sudo yum -y install erlang socat
    curl -s https://packagecloud.io/install/repositories/rabbitmq/rabbitmq-server/script.rpm.sh | sudo bash
    sudo yum install rabbitmq-server -y
    sudo systemctl start rabbitmq-server
    sudo systemctl enable rabbitmq-server
    sudo systemctl status rabbitmq-server
    sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
    sudo rabbitmqctl add_user test test
    sudo rabbitmqctl set_user_tags test administrator
    sudo systemctl restart rabbitmq-server

SSH into the server and check if user data script was provisioned properly, also check status of the service.

    ssh -i awskeypair.pem centos@<public_ip_of_instance>
    sudo -i
    curl http://169.254.169.254/latest/user-data
    systemctl status rabbitmq-server

*Give sometime for the services to be provisioned properly.*

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/rabbitmqlogin.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/rabbitmqstatuscheck.png)

Validated all 3 instances are up and running on the EC2 Console

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/3instancerunningverification.png)

# Step 4

Navigate to Route 53,  click Create hosted zone to create private hosted zones by updating the private IP of our just created instances.

Create vprofile.in Private Hosted zone in Route53. we will pick Default VPC in N.Virginia region.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/hostedzoneandrecords.png)

Create record for each of the backend services, placing their private IP and name in the record within the created hosted zone vprofile.in. 
These records will be used on our Tomct service in the next step

# Step 5

We will be using ubuntu 18 instance for the Tomcat server, and bootstrap with the upon lunch using tomcat_ubuntu.sh in the userdata section. 
Remember to use the same keypair and the security group we created for the app.
Also Add Inbound rule to vprofile-app-SG on port 22 from My IP to be able connect via ssh into the server.

    Name: vprofile-app01
    Project: vprofile
    AMI: Ubuntu 18.04
    InstanceType: t2.micro
    SecGrp: vprofile-app-SG
    UserData: tomcat_ubuntu.sh

## user data for Tomcat

    #!/bin/bash
    sudo apt update
    sudo apt upgrade -y
    sudo apt install openjdk-8-jdk -y
    sudo apt install tomcat8 tomcat8-admin tomcat8-docs tomcat8-common git -y
    
Again allow few minutes for the service to be provisioned.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/appserverloginand%20statuscheck.png)

# Step 6: Build Our Application from source code

Clone the repository to you local system from which an artifact will be created. 

Next update the application properties /src/main/resources directory for below lines to include the CNAME configured on Route 53 
which are `db01.vprofile.in`, `rmq01.vprofile.in` and `mc01.vprofile.in` in the file src/main/resources/application.properties file.

    #JDBC Configutation for Database Connection
    jdbc.driverClassName=com.mysql.jdbc.Driver
    jdbc.url=jdbc:mysql://db01.vprofile.in:3306/accounts?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
    jdbc.username=admin
    jdbc.password=admin123

    #Memcached Configuration For Active and StandBy Host
    #For Active Host
    memcached.active.host=mc01.vprofile.in
    memcached.active.port=11211
    #For StandBy Host
    memcached.standBy.host=127.0.0.2
    memcached.standBy.port=11211

    #RabbitMq Configuration
    rabbitmq.address=rmq01.vprofile.in
    rabbitmq.port=5672
    rabbitmq.username=test
    rabbitmq.password=test

    #Elasticesearch Configuration
    elasticsearch.host =192.168.1.85
    elasticsearch.port =9300
    elasticsearch.cluster=vprofile
    elasticsearch.node=vprofilenode

Build our artifact by running the command from your root directory where the `pom.xml` file is located

    mvn install

After a successful build we should have a Target directory in our current directory where we have our `.war` file will be found.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/Artifactbuildsuccess.png)

# Step 7 : Upload our Application to S3 bucket

Create an IAM user for authentication on AWS CLI and attach s3 full access policy to the user

Configure aws cli to use IAM user credentials.

    aws configure
    AccessKeyID: 
    SecretAccessKey:
    region: us-east-1
    format: json

Upload the built artifact to s3 bucket using AWS CLI 

    aws s3 mb s3://vprofile-artifact-storage-tb

Change directory to target directory then copy the artifact to bucket you just created. 
Then verify by listing objects in the bucket or open the console to see the s3 created with the object 

    aws s3 cp vprofile-v2.war s3://vprofile-artifact-storage-tb/vprofile-v2.war
    aws s3 ls s3://vprofile-artifact-storage-tb/

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/s3createdartifact.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/s3createdartifactclicheck.png)

In order to upload our artifact from s3 to tomcat server, we create a role for our EC2 and attach s3fullaccess permission, 
then attach the IAM role to vprofile-app01 instance

    Type: EC2
    Name: vprofile-artifact-storage-role
    Policy: s3FullAccess

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/attachiamroletoapp01.png)


Then connect to app01 Ubuntu server, switch to root user and verify tomcat8 is running.

    ssh -i awskeypair.pem ubuntu@<public_ip_of_server>
    sudo -i
    systemctl status tomcat8

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/tomcatstatusrunning.png)

Delete ROOT which have the default application under /var/lib/tomcat8/webapps/directory . 
Please ensure to stop the Tomcat server deleting it. follow the command

    cd /var/lib/tomcat8/webapps/
    systemctl stop tomcat8
    rm -rf ROOT

# Step 8: Upload our Application to S3 bucket

Download the artifact from s3 through aws cli . First, install aws cli, then download the artifact to the /tmp/ directory.

Copy the artifact to /var/lib/tomcat8/webapps/ROOT.war to become the default application

Because this is the default app directory, Tomcat will extract the compressed file from there. 

Then restart tomcat8 service

    apt install awscli -y
    aws s3 ls s3://vprofile-artifact-storage-tb
    aws s3 cp s3://vprofile-artifact-storage-tb/vprofile-v2.war /tmp/vprofile-v2.war
    cd /tmp
    cp vprofile-v2.war /var/lib/tomcat8/webapps/ROOT.war
    systemctl start tomcat8

Verify that the `application.properties` file has the latest changes made.

    cat /var/lib/tomcat8/webapps/ROOT/WEB-INF/classes/application.properties

To validate network connectivity from server install telnet.

    apt install telnet
    telnet db01.vprofile.in 3306

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/telnetserviceconnitioncheckdb.png)

# Step 9: Setup ELB with HTTPS [certificate from (ACM)Amazon certificate manager

Navigate back to EC2 on the AWS console and to the load balancer service to setup load balancer but first create a target group for the app01 instance 

    Target type: instance 
    Target Grp Name: vprofile-app-TG
    protocol-port: HTTP:8080
    healthcheck path : /login
    Advanced health check settings
    Override: 8080
    Healthy threshold: 3

Click next and you should have the target group created. Click on Target and then click on edit to updated the edit to add app01 instance to the registered targets and save.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/targetgroupsetup.png)

    Name: vprofile-prod-elb
    Type: Internet Facing
    Select all the AZs
    SecGrp: vprofile-elb-secGrp
    Listeners: HTTP, HTTPS
    Select the certificate for HTTPS

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/ELBcreatedsuccess.png)

# Step 10: Map ELB Endpoint to website name in Amazon route 53 and Verify

Then create the application load balancer and copy the URL to the domain name on Route 53.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/cname4elb.png)

Verify you can access the application on URL setup. 

https://vprofileapp.tabnob.net/login

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/vprofileapplogin.png)

Enter the login details `admin_vp` for both username and password


![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/apploginpage.png)

### Verify all service 


Rabbitmq service verification

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/rabbitmqpagevalidateion.png)

Database service verification

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/dbpagevalidation.png)


Memcached service verification

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/memchachepagevalidation.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/devopspagevalidation.png)

# Step 11: Configure Auto Scaling Group for Application Instance

To set up autoscaling group for tomcat EC2 instance, we will create an AMI from the vprofile-app01 instance.

The image created will be use to launch the configuration for the autoscaling group based on the AMI created.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/vprofileAMI.png)

Lunch configuration for autoscaling group

    Name: vprofile-app-LT
    AMI: vprofile-app-image
    InstanceType: t2.micro
    IAM Profile: vprofile-artifact-storage-role
    SecGrp: vprofile-app-SG
    KeyPair: vprofile-prod-key

Create our ASG.

    Name: vprofile-app-ASG
    ELB healthcheck
    Add ELB
    Min:1
    Desired:1
    Max:4
    Target Tracking-
    CPU Utilization 50

Terminate the existing app instance, ASG will create a new one using Lunch configuration we created.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-%203:%20Lift%20And%20Shift%20Web%20Application%20to%20AWS/images/ASG.png)

Validate login, check all services again 
### Ensure to delete all the instances once you are done to avoid cost.


# Automated Setup of A Multi Tier Web App
## Vprofile-Project2: 

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/vprofile-project2/images/Architectdaigram1.png)

# Prerequisite

To complete this project you should have your lab set-up with the appropriate tools.

 - Git, Bash or any Code editor of choice
 - Oracle VirtualBox
 - Install Vagrant, and Vagrant Plugins.

# Project Architecture

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/vprofile-project2/images/Architectdaigram2.png)

This is a follow up project on the my last project: Multi Tier Web Application Stack Setup Locally but this time we will be automating 
the entire process with bash scripts and vagrant to bring up our VMs and provision the services with just one command.

Write your own bash script or inspect the script as contained in the project repo and explained in detail below. 
The Order of execution are also detailed in the vagrantfile

# Bash Script for the DataBase

All of the commands ran manually will now be scripted in an executable file called `mysql.sh` to provision the database service. 
Each line of script are command executed within the `db01` VM created through vagrant.

    #!/bin/bash
    DATABASE_PASS='admin123'
    sudo yum update -y
    sudo yum install epel-release -y
    sudo yum install git zip unzip -y
    sudo yum install mariadb-server -y

## Starting & Enabling mariadb-server

    sudo systemctl start mariadb
    sudo systemctl enable mariadb
    
    cd /tmp/
    git clone -b local-setup https://github.com/devopshydclub/vprofile-project.git

## Restore the dump file for the application

    sudo mysqladmin -u root password "$DATABASE_PASS"
    sudo mysql -u root -p"$DATABASE_PASS" -e "UPDATE mysql.user SET Password=PASSWORD('$DATABASE_PASS') WHERE User='root'"
    sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')"
    sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User=''"
    sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%'"
    sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
    sudo mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
    sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123'"
    sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123'"
    sudo mysql -u root -p"$DATABASE_PASS" accounts < /tmp/vprofile-project/src/main/resources/db_backup.sql
    sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"

## Restart mariadb-server

    sudo systemctl restart mariadb

## Starting the firewall and allowing the mariadb to access from port no. 3306

    sudo systemctl start firewalld
    sudo systemctl enable firewalld
    sudo firewall-cmd --get-active-zones
    sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
    sudo firewall-cmd --reload
    sudo systemctl restart mariadbBash Script for Memcached

# Bash Script for memcached service

This bash script named `memcache.sh` will provision the memcached service executed by vagrant when the `mc01 server` is created. 
All of the instructions are also contained in the Vagrantfile.

    #!/bin/bash
    sudo yum install epel-release -y
    sudo yum install memcached -y
    sudo systemctl start memcached
    sudo systemctl enable memcached
    sudo systemctl status memcached
    sudo memcached -p 11211 -U 11111 -u memcached -d

# Bash Script for Rabbitmq

Create a file named `rabbitmq.sh` . The below bash script will create the rabbitMQ service.

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

# Bash Script for Tomcat

Create a file named `tomcat.sh`, the bash script will provision Tomcat service and will build and deploy in the `app01` server for our application.

    TOMURL="https://archive.apache.org/dist/tomcat/tomcat-8/v8.5.37/bin/apache-tomcat-8.5.37.tar.gz"
    yum install java-1.8.0-openjdk -y
    yum install git maven wget -y
    cd /tmp/
    wget $TOMURL -O tomcatbin.tar.gz
    EXTOUT=`tar xzvf tomcatbin.tar.gz`
    TOMDIR=`echo $EXTOUT | cut -d '/' -f1`
    useradd --shell /sbin/nologin tomcat
    rsync -avzh /tmp/$TOMDIR/ /usr/local/tomcat8/
    chown -R tomcat.tomcat /usr/local/tomcat8

    rm -rf /etc/systemd/system/tomcat.service

    cat <<EOT>> /etc/systemd/system/tomcat.service
    [Unit]
    Description=Tomcat
    After=network.target

    [Service]

    User=tomcat
    Group=tomcat

    WorkingDirectory=/usr/local/tomcat8

    #Environment=JRE_HOME=/usr/lib/jvm/jre
    Environment=JAVA_HOME=/usr/lib/jvm/jre

    Environment=CATALINA_PID=/var/tomcat/%i/run/tomcat.pid
    Environment=CATALINA_HOME=/usr/local/tomcat8
    Environment=CATALINE_BASE=/usr/local/tomcat8

    ExecStart=/usr/local/tomcat8/bin/catalina.sh run
    ExecStop=/usr/local/tomcat8/bin/shutdown.sh


    RestartSec=10
    Restart=always

    [Install]
    WantedBy=multi-user.target

    EOT

    systemctl daemon-reload
    systemctl start tomcat
    systemctl enable tomcat

## Clone source code 

    git clone -b local-setup https://github.com/devopshydclub/vprofile-project.git
    cd vprofile-project

## Build the application

    mvn install
    systemctl stop tomcat
    sleep 60

## Remove default tomcat
    rm -rf /usr/local/tomcat8/webapps/ROOT*

## Copy the artifact
    cp target/vprofile-v2.war /usr/local/tomcat8/webapps/ROOT.war

## Resarting tomcat
    systemctl start tomcat
    sleep 120

    cp /vagrant/application.properties /usr/local/tomcat8/webapps/ROOT/WEB-INF/classes/application.properties
    systemctl restart tomcat

# Bash Script for Nginx

Again, create a file named `nginx.sh` for the `web server` (web01). The bash script will provision nginx service which serves both as our web 
frontend interface and load balancer that will forward requests to our backend application.

## Adding repository and installing nginx  

    apt update
    apt install nginx -y

    cat <<EOT > vproapp
    upstream vproapp {

     server app01:8080;

    }

    server {

      listen 80;

    location / {

      proxy_pass http://vproapp;

    }

    }

    EOT


    mv vproapp /etc/nginx/sites-available/vproapp
    rm -rf /etc/nginx/sites-enabled/default
    ln -s /etc/nginx/sites-available/vproapp /etc/nginx/sites-enabled/vproapp

## Starting nginx service and firewall

    systemctl start nginx
    systemctl enable nginx
    systemctl restart nginx

Now that we have all the services completely scripted we will proceed to ensure they are contained in the vagrantfile. 
Our vagrantfile should look like below. Note that the order explicitly creates the the DB first, then Memcached, followed by RabbitMQ, 
Tomcat and finally our web app, nginx. They are all contained in the repo.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/vprofile-project2/images/vagrantfile.png)

Edit `application.properties` File

Ensure to edit the `properties.application` file with the right details used to set up all the services in the bash scripts

    #JDBC Configutation for Database Connection
    jdbc.driverClassName=com.mysql.jdbc.Driver
    jdbc.url=jdbc:mysql://db01:3306/accounts?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull
    jdbc.username=admin
    jdbc.password=admin123

    #Memcached Configuration For Active and StandBy Host
    #For Active Host
    memcached.active.host=mc01
    memcached.active.port=11211
    #For StandBy Host
    memcached.standBy.host=127.0.0.2
    memcached.standBy.port=11211

    #RabbitMq Configuration
    rabbitmq.address=rmq01
    rabbitmq.port=5672
    rabbitmq.username=test
    rabbitmq.password=test

    #Elasticesearch Configuration
    elasticsearch.host =192.168.1.85
    elasticsearch.port =9300
    elasticsearch.cluster=vprofile
    elasticsearch.node=vprofilenode

Procceding, clone the repository

    git clone https://github.com/devopshydclub/vprofile-project.git

Change directory to Vagrant directory and cd to the automation_provisioning directory. 
Then run the vagrant up command

    cd vprofile-project/vagrant/Automated_provisioning
    vagrant up

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/vprofile-project2/images/VMs%20up.png)

Observe in your oracle virtualbox UI that the VMs are being created by vagrant in the order we specified in the vagrantfile, 
and the bash script are executed in various VMs immediated they are up, running and stable. (Bootstrap). 
*Note: this will take sometime depending on your internet speed*.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/vprofile-project2/images/VMprovisioned.png)

# Application Validation

Now that all VMs and services are provisioned, we need to validate their connectivity and working state. 
To do this open a browser and type the IP address of the web app (you can get this from the vagrantfile or by just simply using 
the name assigne to the web app) in our case `web01`. This should open the login page of our application confirming our frontend is working.

Type; http://web01

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/vprofile-project2/images/loginpage.png)

Next check that Frontend(via nginx) communicate with Backend server and services. 
To do this; Type user name `admin_vp` as login details for both username and password. 
if you are successful then that validation is complete.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/vprofile-project2/images/tomcat%20validated.png)

Next validate RabbitMq our queing agent service is also provisioned correctly and working by clicking on the Rabbitmq button. 
You should have below display. This indicate that there is connection and communication between `app01` and `rmq01` servers.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/vprofile-project2/images/Rabbitmq.png)

Now lets check DB and memcache services . Go back to home page and Click on ALL USER button, them select any user of your choice. 
his should return data from the DB and insert in Cache the first time you access that information.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/vprofile-project2/images/DBvalidation.png)

Try to access the same information again and you will see it returned from cache instead of DB. Meaning `mc01 server` is provisioned ok and working

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/vprofile-project2/images/cachevalidation.png)

We have created a multi stack App automatically through vagrant with bash scripts successfully and validated its functionlity. 
With this we can re-create the application easily with just one command any time we want. 
Meaning we have our Infrastructure as a code (IaaC). The beauty of automation.

To stop the Stack, run the command on your terminal in the working directory where the vagrantfile is located

    vagrant halt

To check status of the VMs , run the command

    vagrant status

To re-start the VMs again, run the command

    vagrant up

Finally to destroy the VMs, run the command

    vagrant destroy

See you on the next project, Keep learning and practicing.


# Continuous Integration on AWS Cloud

### DevOps Project-6
Project Source: DevOps Project by [Imran Teli](https://www.udemy.com/course/devopsprojects/learn/lecture/23897820#overview)

## Architectural Diagram

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/ProjectArchitecture.png)

### Project Prerequisites:

- AWS Account
- AWS Cloud Services
  1. AWS CodeCommit ( as Version control System Repository)
  2. AWS Codebuild ( to Build services from AWS)
  3. AWS Codedeploy ( as Artifact Deployment Service)
  4. AWS Codeartifact ( Maven Repository for Dependencies)
  5. S3 Bucket ( deploy artifact for storage)
  6. AWS Codepipeline ( Service to integrate all jobs together)
- AWS CLI
- SolarCloud
- Checkstyle

### Flow of Execution

- Login to AWS Account
- Code Commit Setup
    a. Create Codecommit repo
    
    b. Create IAM User with codecommit Policy
    
    c. Generate ssh key locally
    
    d. Exchange keys with IAM User
    
    e. Put Source code from github repo to code commit repository and push
    
- Code Artifact Setup
    a. Create an IAM User with code artifact access
    
    b. Install AWS CLI and configure
    
    c. Export auth token
    
    d. Update settings.xml file in source code top level directory with provide details below
    
    e. Update pom.xml file with repo details
    
- Sonar Cloud Setup
- 
    a. Create sonar cloud account
    
    b. Generate token
    
    c. Create SSM Parameters with sonar details
    
    e. Create Build project
    
    f. Update codebuild role to access SSM parameter store
    
 - Create Notification for SNS
 - Setup Build Project
 
    a. Update pom.xml with artifact version with timestamp
    
    b. Create variables in SSM Parameter store
    
    c. Create build Project
    
    d. Update Codebuild role`to access SSM parameter store
    
  - Create Pipeline
  
    a. Codecommit
    
    b. Test code
    
    c. Build
    
    d. Deploy to s3 bucket
    
 - Test Pipeline


## Step1: AWS CodeCommit Setup


Login to your AWS management console search for CodeCommit service, select repository and create a repository named


      Name: vprofile-code-repo
   

Create an IAM User for the CodeCommit repository created, as well as a policy for the Codecommit that gives full access only to the vprofile-code-repo. 
Name the policy vprofile-code-admin-repo-fullaccess.

To create an IAM, and attaching the policy specifically to the CodeCommit repository use the steps below


      Navigate IAM > click Add user

      IAM User Name: vprofile-code-admin


Next Create a policy to be attached

      service: CodeCommit
      name: vprofile-code-repo
      action: all
      Policy name: vprofile-code-admin-repo-fullaccess

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/codcommitCreatepolicy1.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/codcommitpolicycreated2.png)

Attached the Policy to the IAM User and proceed to create the User.

After the creation of the IAM user we will need to upload SSH public key within the Security credentials section of the newly created user. 
This is done by generating SSH on our local machine and uploading the ssh public key to IAM role Security credentials. **`(SSH public keys for AWS CodeCommit section)`**

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/sshkeygencodecommit.png)

Furthermore we also need to update configuration. Within the .ssh directory create a file name **`.ssh/config` and add ssh host information.**

        Host git-codecommit.us-east-1.amazonaws.com
            User <SSH_Key_ID_from IAM_user>
            IdentityFile ~/.ssh/vpro-codecommit_rsa

Ensure to change file permissions to read and write with this command **chmod 600 config**

Test that the ssh connection to CodeCommit is established and successfully authenticated with the command below.

        ssh git-codecommit.us-east-1.amazonaws.com

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/sshcodecommitvalidated.png)

Next we clone the repository to our local system/machine from Github .
**[Source code](https://github.com/Adutoby/vprofile-project-all)**.**Observe that the url remote origin is git@github.**

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/clonegithubrepo.png)

Convert the repository to CodeCommit repository. To do this, make sure to 
be in the cloned repository and run below commands

        git checkout master
        git branch -a | grep -v HEAD | cur -d'/' -f3 | grep -v master > /tmp/branches
        for i in `cat  /tmp/branches`; do git checkout $i; done
        git fetch --tags
        git remote rm origin
        git remote add origin ssh://git-codecommit.us-east-1.amazonaws.com/v1/repos/vprofile-code-repo
        cat .git/config
        git push origin --all
        git push --tags

    Now observe that the remote origin of the repo has now changed from Github to Codecommit . use command cat .git/config to access it

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/gitoriginconverttocodecommit.png)

Source code branched pushed to Codecommit and tagged accordingly

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/gitpushtocodecommit.png)

Verify to confirm repository contains our code on Codecommit. 
Repo is now ready with all branches present on AWS Codecommit.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/repowithcodeonawscodecommit.png)


## Step2: AWS CodeArtifact Setup and System manager parameter store.


Create CodeArtifact repository for Maven.

      Name: vprofile-maven-repo
      Public upstraem Repo: maven-central-store
      This AWS account (your AWS account ID will be displayed)
      Domain name: visualpath
      and click create repository

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/createcodeartifact1.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/createcodeartifact2.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/createcodeartifact3.png)

Follow the connection instructions as provided in CodeArtifact for the just created **vprofile-maven-repo.**
click on view connection instructions, select you machine type and follow.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/codeartifactconnectioninstruction.png)

IAM user for CodeArtifact will need to be created and configure on AWS CLI with its credentials. 
Give Programmatic access to the user to enable it use AWS CLI and download credentials file.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/createcodeartifactuserpolicyandkey.png)

Run **`aws configure`** to setup the credentials .

        aws configure # enter your IAM user credentials

The command below is from the connection instructions get token section, copy and enter into your terminal or IDE using **AWS CLI**

        export CODEARTIFACT_AUTH_TOKEN=`aws codeartifact get-authorization-token --domain visualpath --domain-owner <aws account ID> --region us-east-1 --query authorizationToken --output text`

Used this below command to print the token to your terminal

        echo $CODEARTIFACT_AUTH_TOKEN

Proceed to update pom.xml and setting.xml file with correct url’s that were suggested in the connection instruction.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/codeartifactconnectioninstruction.png)

Then push both files to CodeCommit. Use below git commands to accomplish that

      git add .
      git commit -m "message" (add you commit message)
      git push origin ci-aws


## Step3: Setup SonarCloud


Create an Account if you don't already have one on the official web page [sonarcloud.io](https://sonarcloud.io)

Click on the account avatar > My account > Security. Give token a name **vprofile-sonartoken** . > click on generate.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/Sonarcloudtokengenerated.png)

Next we create a project,click on the + sign on the top right corner of the web page >Analyze Project >Create Project manually . 
Note down below information they will be used in our build job. Note: Create your Unique organization and project-key but remember store them safely.

        Organization: tabnob-projects
        Project key: tabnob-vprofile-repo
        sonarcloud url: sonarcloud.io
        generated token: <>

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/Sonarcloudreadypage.png)


## Step4: AWS Systems Manager Parameter Store for sonar details


Create parameters with the variables below.

        CODEARTIFACT_TOKEN   Type:SecureString	Value: (value of echo $CODEARTIFACT_AUTH_TOKEN)
        HOST                 Value: https://sonarcloud.io type: string
        ORGANIZATION         Value:tabnob-projects type: string
        PROJECT              Value: tabnob-vprofile-repo type: string
        SONARTOKEN           Value: vprofile-sonartoken geenrated token SecureString

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/AWSSSMparameterstoredetails.png)


##  Step5: AWS CodeBuild for SonarQube Code Analysis


Navigate CodeBuild on your AWS management console and click Create Build Project. 
This preceding steps are similar to Jenkins Job.

        ProjectName: Vprofile-Build
        Source: CodeCommit
        Branch: ci-aws
        Environment: Ubuntu
        runtime: standard:5.0
        select New service role: codebuild009-Vprofile-Build-service-role 
        Insert buildspec content from directory aws-files/sonar_buildspec.yml, cat its content and past in the insert build command tab
        under Logs > GroupName: vprofile-buildlogs
        StreamName: sonarbuildjob

Ensure **`sonar_buildspec.yml`** file parameter store sections are updated with the exact names given in SSM Parameter store. 
Just as you have named them

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/AWSSSMparameterstoredetails.png)

After the build project is created, we need to add a policy to the service role created for this Build project. 
To do this Click on Edit > environment and copy the role name displayed and click cancel.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/codebuildcopyrolenamefromenvironment.png)

Navigate to IAM > click roles > paste the role in the search bar to find the copied role from our build project, 
the role should appear, click on it and edit to attach policy.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/editcodebuildrole1.png)

Choose the service (System Manager), In List access level select the Describeparameter box

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/editcodebuildrole2.png)

Also in the Read access level select the get parameter boxes as indicated below

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/editcodebuildrole3.png)

Review and give your policy a name “vprofile-sonarparameteraccess” and then click create policy.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/editcodebuildpolicy4.png)

Attach the just created policy to the CodeBuild project role.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/editcodebuildpolicyattachedsuccess4.png)

Now that CodeBuild has the right permission, proceed to build the project by clicking the 
**Start Build** on the top right coner of the CodeBuild UI on your management console.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/Codebuildsuccessful1.png)

Once build is completed, check for published results in SonarCloud. 
We have being able to CodeBuild, send and analyze our published result to SonarCloud.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/Sonarcloudresultsuccessful.png)


## Step6: AWS CodeBuild for Build Artifact


On your AWS Console, navigate back to CodeBuild then click the Create Build Project. 
Again this proceeding steps are similar to Jenkins Job just like we did in the SonarQube code analysis.

        ProjectName: Vprofile-Build-Artifact
        Source: CodeCommit
        Branch: ci-aws
        Environment: Ubuntu
        runtime: standard:5.0
        select New service role: codebuild010-Vprofile-Build-Artifact-service-role
        Insert buildspec: (switch to editor) paste the content from directory aws-files/build_buildspec.yml, cat its content and past in the insert build command tab
        under Logs > GroupName: vprofile-buildlogs
        StreamName: artifactbuildjob

After you have created build project, you will need to update the new service role policy and to do so, follow the steps

Click on the created build artifact project > click on edit and select the environment option. 
Copy the service role name(see screenshot direction below)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/editcodebuildartifactrole1.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/editcodebuildartifactrole2.png)

Next navigate to IAM > select roles > search for the service role we just copied > Click on it > click the add permission > click the attach policy option.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/editcodebuildartifactrolepolicy3.png)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/editcodebuildartifactrolepolicy4.png)

Next search and use the policy used for the sonaraccessparameter created in the last step. Select it and attach it to the service role. 
This should enable the list access level and read access level permissions needed to build our job. 
(**refer back to step5 to see full Read access level and List access parameter permissions**)

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/editcodebuildartifactrolepolicy5.png)

With the policy updated, it is time to build our project. Navigate back to your CodeBuild, 
select the **Vprofile-Build-Artifact** project and click on **Start Build**

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/Buildartifatesuccessful.png)


## Step7: AWS CodePipeline and Notification with SNS


Navigate on your AWS console to SNS service and create an SNS topic. next is to subscribe to a topic with your email.

        SNS Topic Name: vprofile-pipeline-notification
        Topic Type: Standard
        subcription protocol: email

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/SNStopic1.png)

Once subscribed with email provided, go to your inbox on the provided email and confirm the notification subscription.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/SNStopicsubcription1.png)

Next is to Create CodePipeline with below details

        Name: vprofile-CI-Pipeline
        SourceProvider: Codecommit
        branch: ci-aws
        Change detection options: CloudWatch events
        Build Provider: CodeBuild
        ProjectName: vprofile-Build-Aetifact
        BuildType: single build
        Deploy provider: Amazon S3
        Bucket name: vprofile98-build-artifact
        object name: pipeline-artifact

When pipeline is creating stop the pipeline and add Test Stages to your pipeline by clicking Add Stage after the Source Stage, 
Name it **Test** and follow the details

        Action Name: Sonar-Code-Analysis
        Action Provider: AWS CodeBuild
        input aritfact: SourceArtifact
        Project Name Vprofile-Build
        Build Type: Select the Single Build
        click done

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/codepipelineaddstageTest.png)

Add another stage after the Build Stage, Name it **Deploy**, and use the details below

        Action Name: Deploy-To-S3
        Action provider: Amazom s3
        Region: Same region of codebuild
        input artifact: BuilArtifact
        Deployment Path: pipeline-artifact (this is the folder name created in your s3 bucket for the pipeline)
        check the Extract file before deploy option
        Click Done

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/codepipelineaddstagedeploy.png)

Ensure to have save the pipeline after adding the Test and Deploy stages.

Next step is setting up the Notifications. Navigate to setting in the CodePipeline > click the notification option > click the Create Notification rules.

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/codepipelinesnsnotificationrulesetup.png)

We are all set to run our Pipeline now , click on **Release Change** and the Pipeline should begin to run

![](https://github.com/Adutoby/DevOps-Projects-AWS/blob/master/Project-6%3A%20Continuous%20Integration%20on%20AWS%20Cloud/images/codepipelinesuccessfullydeployed.png)

Pipeline was successful as all 4stages were returned succeeded. Great Job!! to make it this far. One last step, to validate that our Pipeline will trigger 
a job when code change happens and committed to the CodeCommit which is our source.


## Step8: Validate CodePipeline

To validate our CodePipeline is setup properly for an automatic build job as intended, lets make some changes to the README.md file in the 
root directory of our source code. Add the changes to stage them, commit the changes and push to the git repository which is CodeCommit. 
Once changes are pushed, CloudWatch will detect the changes and trigger a Pipeline job.

This brings us to the end of the project where we have streamlined our CI processes, reduced several overhead associated with the management 
and maintenance of several servers to now having our Continuous Integration process executed within the AWS platform using the aforementioned 
AWS services for effective, efficient, reliable and scalable process on a single platform plus the storage of our Artifacts to S3 bucket.

Thank you for reading. please react and send in your comments and don’t forget to connect with me on LinkedIn [HERE](https://www.linkedin.com/in/oluwatobiloba-adu/).

See you on the next project, Keep learning and practicing.

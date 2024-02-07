

![image](/images/cover22.png)


### PROJECT-OVERVIEW

In this project I build a CI/CD pipeline tailored for Terraform projects. By automating the building and deployment of infrastucture changes, the piplenine enhances speed, reliability and consistency across environments. 

#### TECH STACK USED:
- **Terraform** : To provision our resources on AWS
- **Docker** : Jenkins has a Docker image that can be used out of the box to run a container with all relevant dependencies for Jenkins. 
- **JENKINS** :  Jenkins as our continuous integration server, to automate the building and deployment of code changes.
- **AWS EC2** (*optional*) : I used an EC2 instance to run Docker because Docker Desktop was consuming a lot of RAM, feel free to use Docker Desktop and run the project locally if your computer allows.
- **GITHUB** : We use Github Webhooks to ensures that Jenkins is automatically triggered whenever changes are pushed to the repository, initiating the CI/CD process.
- **Slack** : We configure Jenkins to send a notification to Slack updating the team on the build and deployment status. 

### SETTING UP THE ENVIRONMENT

1. Create an EC2 instance, make sure that it is a `t2.medium` or run Docker Desktop. In my case, I used an EC2 instance.

![image](/images/Screenshot_24.png)

2. Next, you need to have Docker running in your instance or Docker desktop. To install Docker in your instance run `sudo yum install docker -y`

![image](/images/Screenshot_25.png)

3. Now that Docker is installed you need to start Docker. Run `sudo service docker start`



![image](/images/Screenshot_26.png)

#### Now let's create our Dockerfile

4. Create a directory and name it `terraform-with-cicd` and `cd` into the directory
5. Inside the directory create a file and name it `Dockerfile`
6. Run `sudo vi Dockerfile` and add this script

```
 # Use the official Jenkins base image
 FROM jenkins/jenkins:lts

 # Switch to the root user to install additional packages
 USER root

 # Install necessary tools and dependencies (e.g., Git, unzip, wget, software-properties-common)
 RUN apt-get update && apt-get install -y \
     git \
     unzip \
     wget \
     software-properties-common \
     && rm -rf /var/lib/apt/lists/*

 # Install Terraform
 RUN apt-get update && apt-get install -y gnupg software-properties-common wget \
     && wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | tee /usr/share/keyrings/hashicorp-archive-keyring.gpg \
     && gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint \
     && echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/hashicorp.list \
     && apt-get update && apt-get install -y terraform \
     && rm -rf /var/lib/apt/lists/*

 # Set the working directory
 WORKDIR /app

 # Print Terraform version to verify installation
 RUN terraform --version

 # Switch back to the Jenkins user
 USER jenkins

```


![image](/images/Screenshot_27.png)

#### EXPLAINING THE DOCKERFILE

- `FROM jenkins/jenkins:lts` : This line specifies the base image for your Dockerfile, this image will be pulled from Docker Hub since you do not have it locally.

- `USER root` : This command switched to the root user within tthe Docker image. This is done to perform actions as a root user, for permissions such as installing packages.

- The below script installs necessary tools and dependencies. The `apt-get update` command refreshes the package list and `apt-get install` installs the packages `(git, unzip, wget, software-properties-common)`. The `&&` is used to chain commands and `rm -rf /var/lib/apt/lists/*` removes unnecessary package lists, helping to reduce the size of the Docker image.

 ```
RUN apt-get update && apt-get install -y \
    git \
    unzip \
    wget \
    software-properties-common \
    && rm -rf /var/lib/apt/lists/*

```
- The next script installs Terraform. It follows similar steps as the script above.

```
RUN apt-get update && apt-get install -y gnupg software-properties-common wget \
    && wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | tee /usr/share/keyrings/hashicorp-archive-keyring.gpg \
    && gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint \
    && echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | tee /etc/apt/sources.list.d/hashicorp.list \
    && apt-get update && apt-get install -y terraform \
    && rm -rf /var/lib/apt/lists/*

```
- `WORKDIR /app` : This line sets the working directory to /app. This is where you will enter the container.

- `RUN terraform --version` : This command prints version of Terraform to the console, allowing you to verify that the installation was successful.

- `USER jenkins` : This command switches the jenkins user returning to a lower privilegelevel. This good for security practice to minimize the risk of running processes as the root user within the container.

#### BUILDING AND RUNNING THE DOCKER IMAGE

Make sure that you are inside the folder containing the Dockerfile. The content of the build is sent to the Docker daemon during the build process and serves as the source for building the Docker image.

- To build the custom jenkins image run `docker build -t jenkins-server .` the " `.` " ant the end is the current directory where the Dockerfile is.


![image](/images/Screenshot_28.png)

- Run the image into a docker container `docker run -d -p 8080:8080 --name jenkins-server jenkins-server 
` This will output a hash data.

- Check that the container is running `docker ps`

![image](/images/Screenshot_30.png)

- Copy your Public IP address and access the Jenkis server from the browser. `public-ip/8080` or `localhost:8080` if you are running Docker Desktop.

![image](/images/Screenshot_31.png)

- Run `/var/jenkis_home/secrets/initialAdminPassword` to get the Jenkins password, enter it and follow the flow of the website, enter all required information.

![image](/images/Screenshot_32.png)

![image](/images/Screenshot_33.png)

![image](/images/Screenshot_34.png)

### SETTING UP JENKINS FOR TERRAFORM CI/CD

We will first Set up a Git repository with Terraform code. The use case we will satisfy is that:

- Your terraform code has an existing set of resources that it creates in your preffered cloud provider.

- As a DevOps Engineer you intend to create an additional resources by updating the code base with the new resource that needs to be created.

We are using this code https://github.com/dareyio/terraform-aws-pipeline you can use any Terraform code. 

1. Clone the code base into your local machine. This is where you will create additional resources.


![image](/images/Screenshot_15.png)

2. In the `provider.tf` file create an s3 bucket in your AWS account ad push your latest chnages to GitHub then run `Terraform init, Terraform plan, Terraform apply`

![image](/images/Screenshot_14.png)

![image](/images/Screenshot_19.png)

### INSTALL PLUGINS FOR JENKINS

1. To connect your GitHub repo to Jenkins, you first need to install "**GitHub Intergration**" plugin.

Navigate to *Manage Jenkins --> Plugins --> Available plugins --> GitHub Intergration*
![image](/images/Screenshot_35.png)

2. Install the "**Terraform**" plugin, this plugin enables seamless intergration of Terraform into Jenkins pipelines.

Navigate to *Manage Jenkins --> Plugins --> Available plugins --> Terraform*
![image](/images/Screenshot_36.png)

3. Install the "**AWS Credentials**" for securely managing and using AWS credentials within Jenkins.

Navigate to *Manage Jenkins --> Plugins --> Available plugins --> AWS Credentials*
![image](/images/Screenshot_37.png)

### CONFIGURE GITHUB CREDENTIALS IN JENKINS

- Jenkins needs to know how to connect to GitHub, in the case that a repository is private, it won't know how to access the repository. Hence you need to store GitHub credentials in Jenkins.

In Github navigate to your *profile --> Click on "Settings" --> scroll down to "Developer Settings" --> Tokens(classic)*
![image](/images/Screenshot_38.png)

Generate a new token

![image](/images/Screenshot_39.png)
![image](/images/Screenshot_40.png)

Copy the access token and save it in notepad for use later.

![image](/images/Screenshot_41.png)


In Jenkins, navigate to *Manage Jenkins --> Credentials --> global --> Add credentials*
![image](/images/Screenshot_42.png)


Select username and password. Use the Access token generated earlier on Github as your password and add your Github username under `username`

![image](/images/Screenshot_43.png)

In the credential section you will be able to see the created credential.

![image](/images/Screenshot_44.png)

### CONFIGURE AWS CREDENTIALS IN JENKINS


Assuming you have an IAM user, access your AWS credentials by going to the console and navigate to the directory where your credentials are stored.

From the directory --> *cd .aws --> ls -->cat credentials*
![image](/images/Screenshot_45.png)

Now you need to create a second credentials for AWS secret and access key. Since you installed the **"AWS Credentials"** plugin earlier, you will be able to see it in the drop-down menu. Scroll down and add your IAM access and secret key.

![image](/images/Screenshot_46.png)
![image](/images/Screenshot_47.png)


#### SETUP MULTIBRANCH PIPELINE


From the Jenkins dashboard, click on **"New Item"**
![image](/images/Screenshot_48.png)


Give it a name and description

![image](/images/Screenshot_49.png)

Select the type of source of the code and Jenkinsfile

![image](/images/Screenshot_50.png)

Select the credentials to be used to connect to Github from Jenkins and add the repository URL that you forked and cloned earlier.

![image](/images/Screenshot_51.png)


You will see the scanning of the repository for branches and the Jenkisfile.

![image](/images/Screenshot_52.png)

Pipeline run and Console output, **Terraform Apply** failed.

![image](/images/Screenshot_56.png)
![image](/images/Screenshot_57.png)


To fix the Scripts not permitted to use method, go to the Jenkins 
Dashboard, click on `Manage Jenkins`, scroll down to the `Security Tab` and click on `In-Process Script Approval`

![image](/images/Screenshot_58.png)

Click on `Approve`

![image](/images/Screenshot_59.png)

A manual intervention step asking for confirmation before proceeding. If `Yes` is clicked, it runs the `terraform init & apply`

![image](/images/Screenshot_60.png)

### EXTEND THE PIPELINE

Now extend the existing pipeline script to inclued additional stages and improve the existing ones.

1. Create a new branch from the `main` branch 

![image](/images/Screenshot_61.png)
![image](/images/Screenshot_62.png)

2. Scan the Jenkins pipeline so that the new branch can be discovered.

![image](/images/Screenshot_63.png)

3. In the new branch, correct and enhance the "Terraform Apply"
 stage mistakenly contains a `sh terraform aply -out=tfplan` command. Correct this to `sh 'terraform apply tfplan`

![image](/images/Screenshot_64.png)

4. Add logging to track the progress of the pipeline within both `Terraform plan & apply`. Use `echo` command to print messages before and after each execution so that in the console output everyone can understand what is happening at each stage

![image](/images/Screenshot_65.png)
![image](/images/Screenshot_66.png)
![image](/images/Screenshot_67.png)


4. Introduce a new stage in the pipeline script to validate the Terraform configurations using `terraform validate`. The purpose of this stage is to validate the syntax, consitency, and correctness of Terraform configuration files in the directory it is run.

![image](/images/Screenshot_68.png)

5. Introduce a **"Cleanup Stage"** that runs regardless of whether previous stages succeeded or failed. This stage includes commands to clean up any temporary files or state that the pipeline may have created.


![image](/images/Screenshot_70.png)

Go back to your `terraform-cicd` multibranch pipeline, initiate another build and move your cursor to the Terraform Apply stage and click on 'Yes' to apply changes.

You will notice that `Terraform Apply` has been skipped and that is because it has failed. Now because we introduced the `Cleanup` stage the build did not stop, it skipped to the next stage.

The reason that it has skipped is because we are on the `refactor` branch and should be on `main` branch

![image](/images/Screenshot_73.png)
![image](/images/Screenshot_72.png)

#

### CREATING SLACK FOR NOTIFICATIONS

6. Add error handling to the pipeline. For instance, if `"Terraform Apply"` fails, the pipeline should be handle this gracefully, perhaps by sending a notification or logging detailed error messages. In my case I am using Slack to send the notifications.

The first thing you need is a `Slack channel` this is where notifications will be sent to the team.

1. Got to https://slack.com/get-started#/createnew
2. Enter your email address and click **Continue**
3. Check your email for a confirmation coe. 
4. Enter your code, then click **Create a Workspace** and follow the prompts.

Enter the name of your company. Click -->`next`

![image](/images/Screenshot_82.png)
![image](/images/Screenshot_83.png)

Now you need to intergrate Slack with Jenkins, go to your profile and under `Manage` click on `Installed Apps` then search for `Jenkins` then click on `Add to Slack`

![image](/images/Screenshot_84.png)
![image](/images/Screenshot_85.png)

 Scroll down you will see that your channel has been added. 
 `#jenkins-build` once you see that click on `Add Jenkins CI Intergration`

![image](/images/Screenshot_86.png)

When you scroll down you will find more instructions.

![image](/images/Screenshot_87.png)

Go back to your Jenkins server and follow the above instructions.

Click on **Manage Jenkins --> Plugins -->Available Plugins --> Slack Notification** plugin and install it.
![image](/images/Screenshot_88.png)
![image](/images/Screenshot_89.png)

Next you need to add credentials for Jenkins to interact with Slack. Go back to Slack and copy the `Intergration Token Credential ID`

![image](/images/Screenshot_91.png)

Then, go back to Jenkins. **Manage Jenkins --> Credentials --> System --> Global credentials**

![image](/images/Screenshot_90.png)

On the drop-down select "Kind" as `Secret text` then paste the `Intergration Token Credential ID` that you copied under "Secret"

![image](/images/Screenshot_92.png)

Then follow the next steps.

![image](/images/Screenshot_93.png)

Scroll down you will see that `Slack` is added then select your workspace and the credentials you added and click on `Test connection`
to test the connection between Slack and Jenkins. If everything is configured correctly, you should see `Success`

![image](/images/Screenshot_95.png)


**Now go back to your Jenkinsfile, you need to write a script for the Slack notification.**


First define your variable at the top of the Jenkinsfile.

![image](/images/Screenshot_96.png)

Then inside the `Post`stage add the script for a Slack notification.

![image](/images/Screenshot_97.png)

Next, run another build. You will see that my build has failed. `Error: Could not launch On-Demand Instances: VpcuLimitExceeded`

![image](/images/Screenshot_98.png)
![image](/images/Screenshot_81.png)

So our build failed because of the Limit, however when you going to the `AWS console`, you will see that other resources are created successfully.

![image](/images/Screenshot_77.png)

Now go back to `Slack` and you will see that the notfication of the build failure was sent to the team.

![image](/images/Screenshot_99.png)

To fix the error you can write a request to AWS to increase the Vpcu Limit, but in this case we will not be doing that, once it has been increased the build should run successfully.

#### DESTROY RESOURCES

Lastly, we need to destroy all the resources that are running. Go to your local system and go to the path of the project. Run `terraform init` then `terraform destroy`

![image](/images/Screenshot_101.png)

Next to 'Enter a value' type `Yes`
![image](/images/Screenshot_102.png)
![image](/images/Screenshot_103.png)

Go back to your `AWS console` to confirm that resources have been destroyed.

![image](/images/Screenshot_80.png)


### CHALLENGES ENCOUNTERED DURING PROJECT

1. I kept on getting an EKS permissions error, even though my user had full `AdministratorAccess` permissions. 

- Solution : I had to change the region that I was using.

2. Writting declarative scripts to extend my pipeline

- Solution : Used the documentation and alot of googling to understand the syntax.

# THE END !!



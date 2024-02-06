

![image](/images/cover22.png)


### PROJECT-OVERVIEW

In this project I build a CI/CD pipeline tailored for Terraform projects. By automating the building and deployment of infrastucture changes, the piplenine enhances speed, reliability and consistency across environments. 

#### TECH STACK USED:
- **Terraform** : To build the infrastructure on AWS
- **Docker** : Jenkins has a Docker image that can be used out of the box to run a container with all relevant dependencies for Jenkins. 
- **JENKINS** :  To build the CI/CD pipeline.
- **AWS EC2** (*optional*) : I used an EC2 instance to run Docker because Docker Desktop was consuming a lot of RAM, feel free to use Docker Desktop and run the project locally if your computer allows.
- **GIT & GITHUB** : To push code and GitHub Webhook to trigger Jenkins.
- **Slack** : For post notifications of the CI/CD pipeline. Slack will alert us to let us know if the build failed or was a success. 

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


You will see the scanning of the repository for branches and the Jenkisfile
![image](/images/Screenshot_52.png)

Pipeline run and Console output
![image](/images/Screenshot_56.png)

![image](/images/Screenshot_57.png)

![image](/images/Screenshot_58.png)

![image](/images/Screenshot_59.png)

![image](/images/Screenshot_60.png)

![image](/images/Screenshot_61.png)

![image](/images/Screenshot_62.png)

![image](/images/Screenshot_63.png)

![image](/images/Screenshot_64.png)

![image](/images/Screenshot_65.png)

![image](/images/Screenshot_66.png)

![image](/images/Screenshot_67.png)

![image](/images/Screenshot_68.png)

![image](/images/Screenshot_69.png)

![image](/images/Screenshot_70.png)

![image](/images/Screenshot_71.png)

![image](/images/Screenshot_72.png)

![image](/images/Screenshot_73.png)

![image](/images/Screenshot_74.png)

![image](/images/Screenshot_75.png)

![image](/images/Screenshot_76.png)

![image](/images/Screenshot_77.png)


![image](/images/Screenshot_81.png)

![image](/images/Screenshot_82.png)

![image](/images/Screenshot_83.png)

![image](/images/Screenshot_84.png)

![image](/images/Screenshot_85.png)

![image](/images/Screenshot_86.png)

![image](/images/Screenshot_87.png)

![image](/images/Screenshot_88.png)

![image](/images/Screenshot_89.png)

![image](/images/Screenshot_90.png)

![image](/images/Screenshot_91.png)

![image](/images/Screenshot_92.png)

![image](/images/Screenshot_93.png)

![image](/images/Screenshot_94.png)

![image](/images/Screenshot_95.png)

![image](/images/Screenshot_96.png)

![image](/images/Screenshot_97.png)

![image](/images/Screenshot_98.png)

![image](/images/Screenshot_99.png)

![image](/images/Screenshot_100.png)

![image](/images/Screenshot_101.png)

![image](/images/Screenshot_102.png)

![image](/images/Screenshot_103.png)

![image](/images/Screenshot_80.png)



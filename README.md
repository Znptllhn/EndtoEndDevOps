## End to End DevOps

![resim](https://user-images.githubusercontent.com/60771816/125847344-8446280a-ef18-4301-b66d-036388863a57.png)

### Table of Contents
==============

- [End to End DevOps](#end-to-end-devops)
  * [What is it?](#what-is-it)
  * [What tools were used?](#what-tools-were-used)
  * [What is the lifecycle of this automation process?](#what-is-the-lifecycle-of-this-automation-process)
    + [1- Creating Infrastructure - Terraform](#1--creating-infrastructure---terraform)
    + [2- Creating Environment - Ansible](#2--creating-environment---ansible)
    + [3- Source Control Management - Gitlab](#3--source-control-management---gitlab)
    + [4- Repository - Nexus](#4--repository---nexus)
    + [5- Test - Sonarqube](#5--test---sonarqube)
    + [6- Continuous Integration - Jenkins](#6--continuous-integration---jenkins)
    + [7- Continuous Deployment - ArgoCD](#7--continuous-deployment---argocd)
    + [8- Application - ElvesLibraryApp](#8--application---elveslibraryapp)
  * [How does it look like?](#how-does-it-look-like)

### What is it?
End-to-End DevOps is a project that creates the automation processes from committing a code to a version control system to deploying it to the production environment. It includes creating infrastructure, environment and CI/CD pipelines.
### What tools were used?
- Application: Java Spring Boot Framework
- Source Control Management: Gitlab
- Build: Maven
- Continuous Integration: Jenkins
- Continuous Deployment: ArgoCD
- Repository: Nexus
- Test: SonarQube
- Configuration Management: Ansible
- Infrastructre Provisioning: Terraform
- Container Runtime: Docker
- Container Orchestration: Kubernetes
- Compute: Google Cloud Provider
- Application Server: WebSphere Liberty
- Database: MySQL
- Operating Systems: CentOS 8
### What is the lifecycle of this automation process?
- Change the source code of your application in your IDE locally.
- Commit the change to the Gitlab.
- Jenkins will be triggered by commit.
- Jenkins Continuous Integration Pipeline will be started.
- Source code will be cloned to Jenkins server.
- Source code will be built by Maven.
- Source code will be tested by SonarQube.
- Application will be containerized by Docker.
- New application Docker image version will be pushed to Nexus.
- Image section in rollout.yaml will be replaced with new image version.
- New rollout.yaml will be pushed to Gitlab.
- ArgoCD will acknowledge new rollout.yaml in Gitlab project.
- ArgoCD will deploy application to Kubernetes Cluster using Blue/Green method.
- New application pods will run on Kubernetes while old application pods continue to run via different service.
- Application will run on WebSphere Liberty in a pod and ready to serve clients.
#### 1- Creating Infrastructure - Terraform
First of all, we need to create infrastructure which includes servers, network and firewall rules. We will use Google Cloud Provider for this. You can get 300 $ free balance for 3 months. All you need is a new Google account. This is more than enough for this project. 
We will use Terraform to create servers, network and firewall rules. Since we need to create many servers and configure many things, Terraform script will simplify all these things. Let's start.
First we need to create 2 different images on Gcp. On default, CentOS 8 images don't allow to ssh themselves with using root password. We can use ssh keys, but this will simplify things. Second image is for Ansible. We will install Ansible and take image of that. So after we execute Terraform script. Ansible will be ready to go for configuration management.
We need to create a CentOS 8 image manually for one time. Go to Compute Engine --> Vm Instances --> Create Instances . Select CentOS 8 and ssh into created instance. Use these commands;
```shell
dnf update
```
```shell
sudo su root
cd
passwd root
```
Change to password of your will.
```shell
vi /etc/ssh/sshd_config
```
Change “PermitRootLogin” to “yes” and “PasswordAuthentication” to “yes”, then save and close.
```shell
service sshd restart
```
Go to Compute Engine --> Images --> Create Image . Select the current instance and save the image. Execute these commands for install Ansible.
```shell
dnf epel-release
```
```shell
dnf ansible
```
Again save the image.
Now we need to install Terraform. You can download Windows version of Terraform in [here](https://www.terraform.io/downloads.html "here"). We will install Terraform on our local computer and connect to GCP. After you extract Terraform, don't forget to configure environment parameters. We also need a credentials.json file that Terraform use for connecting to Gcp. I advise you to create a new service account for Terraform on Gcp and from it get the necessary keys for json file. Then open main.tf file on the repository with your favored code editor. You need to replace the "your-ip" lines in the main.tf file with your ip. These firewall rules allow only us to access interfaces like Jenkins. Also be careful for image names. You can change the lines 'image = "centos8-noldor"' with your image names. To execute Terraform script use these;
```shell
terraform init
```
```shell
terraform plan
```
```shell
terraform apply
```
Now check on Gcp. Instances, network and firewall rules should be created. It should look like this;

![resim](https://user-images.githubusercontent.com/60771816/120531164-01345280-c3e7-11eb-8ba5-5e818e7897e5.png)


#### 2- Creating Environment - Ansible
Ssh into ansible-controller instance and type "sudo su" for change user to root. You should do this whenever you connect an instance. Upload IAC_role folder to the server. Don't forget to change server passwords in inventory.txt . In IAC_role folder, execute this command;
```shell
ansible-playbook main.yaml -i inventory.txt
```
It will take a while. Ansible will create baremetal 1 Master, 2 Slave Kubernetes cluster, Jenkins, Gitlab, Nexus and Sonarqube. All configurations will be done and ready to go. 
#### 3- Source Control Management - Gitlab
You can access Gitlab with http://gitlab-external-ip . After you changed the password, you can sign in with root. Then, you should create a project that contains the application source code. To do this, after you create blank project in Gitlab, enter folder in your local computer which contains application and write these one by one in command window;
```shell
git init
git remote add origin your-repo-address
git add .
git commit -m "Initial commit"
git push -u origin master
```
You should be able to see your application code on Gitlab project that you created like this;

![resim](https://user-images.githubusercontent.com/60771816/120544663-54fa6800-c3f6-11eb-9d8e-5d4c15966736.png)

Also we need to create a webhook to integrate Gitlab with Jenkins. Go to project that you created --> Settings --> Webhooks . Write URL area http://jenkins-external-ip:8080/generic-webhook-trigger/invoke . Write Secret token elveslibraryapp . Tick Push Events and add webhook. Untick Enable SSL verification.
#### 4- Repository - Nexus
You can access Nexus with http://nexus-external-ip:8081 . After you login Nexus, click on settings symbol --> Repositories --> Create repository --> docker (hosted) . Configure like this;

![resim](https://user-images.githubusercontent.com/60771816/120084552-df328b80-c0d9-11eb-95e4-ecf72c56a29e.png)

#### 5- Test - SonarQube
You can access SonarQube with http://sonarqube-external-ip:9000 . Default username is admin, password is admin. After you login SonarQube, click on A, right top corner --> My Account --> Security --> Generate Tokens . Save this generated token. We will use it for Jenkins. Then, go to Administration --> Configuration --> Webhooks --> Create . Write URL area http://jenkins-external-ip:8080/sonarqube-webhook/ . 
#### 6- Continuous Integration - Jenkins
You can access Jenkins with http://jenkins-external-ip:8080 . After you login Jenkins interface, install suggested plugins. We need to configure a couple of things.
1. Go to Manage Jenkins --> Manage Plugins --> Available . Install these plugins;
     - Docker Pipeline
     - Pipeline Utility Steps
     - Sonarqube Scanner
     - Generic Webhook Trigger
2. We need to set credentials. Go to Manage Jenkins --> Manage Credentials --> Jenkins --> Global Credentials --> Add Credentials .
     - Gitlab Username with password
     - Nexus Username with password
     - Sonarqube secret text
3. Go to Manage Jenkins --> Configure System --> Sonarqube servers --> Add SonarQube . Configure like this;

![resim](https://user-images.githubusercontent.com/60771816/120084807-f07c9780-c0db-11eb-8971-d6cfb8b9d899.png)

4. Go to Manage Jenkins --> Global Tool Configration --> Sonarqube Scanner --> Add Sonarqube Scanner . Configure like this;

![resim](https://user-images.githubusercontent.com/60771816/120084793-d17e0580-c0db-11eb-8248-dc6232f11169.png)

5. Go to New Item --> Pipeline . Tick Generic Webhook Trigger in Build Triggers section and configure like this; 
```shell
changed_files
$.commits[*].['modified','added','removed'][*]
$changed_files
"app/[^"]+?"
```

![resim](https://user-images.githubusercontent.com/60771816/120084877-81537300-c0dc-11eb-9080-76a830220f1e.png) ![resim](https://user-images.githubusercontent.com/60771816/120084904-cbd4ef80-c0dc-11eb-9e27-ed8bf97416ed.png) ![resim](https://user-images.githubusercontent.com/60771816/120084899-bcee3d00-c0dc-11eb-93a3-a6026040470c.png)

6. Copy the script in repository named jenkinsfile and paste it to pipeline section. Don't forget to set environments in the script. Our Continuous Integration pipeline is ready to go.
#### 7- Continuous Deployment - ArgoCD
Ssh into master instance. Execute this command for edit the argocd-server service;
```shell
kubectl edit svc argocd-server -n argocd
```
Replace ClusterIP value with NodePort like this;

![resim](https://user-images.githubusercontent.com/60771816/120548724-5712f580-c3fb-11eb-99cf-76725de2d7b5.png) ![resim](https://user-images.githubusercontent.com/60771816/120548769-65611180-c3fb-11eb-85b3-3b94bfe08045.png)

Execute this command to see NodePort value;
```shell
kubectl get services -n argocd
```

![resim](https://user-images.githubusercontent.com/60771816/120549263-09e35380-c3fc-11eb-8fe8-727243f36193.png)

Also, we need to create a secret that contains nexus authentication informations. Use this command, replace your nexus password;
```shell
kubectl create secret docker-registry nexus --docker-server=35.222.88.107:8083 --docker-username=admin --docker-password=password
```
You can access with https://slave1-external-ip:nodeport . Get password by running this command;
```shell
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```
After you login ArgoCD, Go to Settings --> Repositories --> Connect Repo Using Https . Write there your Gitlab repo address and username, password. Click connect. Then Go to Application --> New App . Configure like this;

![resim](https://user-images.githubusercontent.com/60771816/120100947-248fa100-c14c-11eb-9aca-1b3e84a8f435.png)

![resim](https://user-images.githubusercontent.com/60771816/120100906-eabe9a80-c14b-11eb-8957-6ca1467ddafd.png)

#### 8- Application - ElvesLibraryApp

Our application is a simple Spring Boot Java web application. It is not a complex work since we don't need one. We focus devops processes of any application only. We have a main page and three definition which user can select. After it selected, application gets the related information of selected object from MySQL database.

Ssh into master instance. We need to insert table contents of MySQL database. Enter MySQL using this command;
```shell
kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -ppassword
```
After you enter MySQL, we need to create database and user for our application;
```shell
CREATE DATABASE elveslibraryapp;
```
```shell
CREATE USER 'elveslibrary'@'%' IDENTIFIED BY 'password';
```
```shell
GRANT ALL PRIVILEGES ON elveslibraryapp.* TO 'elveslibrary'@'%';
```
```shell
USE elveslibraryapp;
```
Make sure tables are created;
```shell
SHOW TABLES;
```
Then use INSERT commands from data.sql file in repository. You should INSERT those contents one by one.
### How does it look like?

Our Jenkins pipeline should look like this after we trigger Build;

![resim](https://user-images.githubusercontent.com/60771816/120790057-1b7f4500-c53b-11eb-8d47-c6010ffb5ed1.png)

Our Nexus repository should look like this after we push the images;

![resim](https://user-images.githubusercontent.com/60771816/120790206-4e293d80-c53b-11eb-8b94-53777c0be260.png)

Our SonarQube should look like this after we test the source code of application;

![resim](https://user-images.githubusercontent.com/60771816/120790282-6a2cdf00-c53b-11eb-8c71-adab1b74dce5.png)

Our ArgoCD deployment should look like this after Jenkins pipeline passed successfully;

![resim](https://user-images.githubusercontent.com/60771816/120790402-98122380-c53b-11eb-85a0-eb27cdbc5f01.png)

And finally our application should look like this;

![resim](https://user-images.githubusercontent.com/60771816/120790469-afe9a780-c53b-11eb-95fa-d9d083e47be7.png)

![resim](https://user-images.githubusercontent.com/60771816/120790501-b9730f80-c53b-11eb-8730-63cc8790ad05.png)

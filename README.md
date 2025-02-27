## Overview
Will build a containerized application in a single-node K8s cluster using [KIND](https://kind.sigs.k8s.io/) (Kubernetes in docker) [kind docs](https://kind.sigs.k8s.io/). Similar to the previous assignment [assignment1](https://github.com/omar0ali/portable-cloud-assignment1)  we will use the same web application from the following repository.

[https://github.com/ladunuthala/clo835_summer2023_assignment1](https://github.com/ladunuthala/clo835_summer2023_assignment1)

We will need ensure to build and push two images to Amazon ECR (**MySQL** and the **Website**) that will be used later after creating the *cluster*.

Will prepare everything locally first before moving to the EC2 Instance. 

>[!NOTE]
>I won't use terraform this time as its optional, I will manually create the ec2 instance and ensure to use `t2.medium` since it worked quite well with KIND before on the previous assignment. 
>
>Will Also use the default settings for VPC, Subnets, Routings and Vockey to simplify the work. Only one thing will ensure to open which are the ports using **Security Groups** 

**Pre-requisites must be installed in EC2 Instance**:
1. Amazon Linux
2. Docker
3. KIND : Kubernetes in docker
4. kubectl
## Steps
#### First Step: Build Images (MySQL, WebSite) And Push To Amazon ECR (Automated in GitHub)
Referring to [main.yaml](https://github.com/omar0ali/portable-cloud-assignment1/blob/main/.github/workflows/main.yml) of the assignment 1 that I worked on, since its already has the code to build and push the images (mysql, flask_app) to Amazon ECR. I copied the part that when it builds the image and push them to amazon ECR.
##### GitHub Action - Build and push images
###### Ensure the following setup in the GitHub Actions Secrets
2. `AWS_ACCESS_KEY_ID`
3. `AWS_SECRET_ACCESS_KEY`
4. `AWS_SESSION_TOKEN`

>[!NOTE]
The containerized application will use **pod**, **replicaset**, **deployment** and **service** manifests. Will expose web application using Service of type **NodePort**. And MySQL will be exposed using a Service of type **ClusterIP**. Lastly, a modification of the config file and another deployment to show a new version of the application.

#### ECR Login EC2 Instance

**First `aws` credentials**
Create the following directory in ec2 instance. And create credentials file within that directory.

```bash
mkdir $HOME/.aws
touch credentials
```

Go to the `AWS Academy Learner Lab` and copy the AWS details for the `AWS CLI` application.

Ensure user logged in.

```bash
aws sts get-caller-identity
```

**Next, `kubectl`**

This is a critical step once we login to the `ec2` instance, will have to login into `aws` and ensure we use the key in `kubectl` will use the following commands to set and ensure when starting the deployment it pulls the image correctly from the registry.

```bash
aws ecr get-login-password --region us-east-1 > ecr-pass.txt
```

```bash
kubectl create secret docker-registry ecr-secret --docker-server=<aws-account-id>.dkr.ecr.us-east-1.amazonaws.com --docker-username=AWS --docker-password=$(cat ecr-pass.txt)
```
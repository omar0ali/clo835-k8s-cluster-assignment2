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
Referring to [main.yaml](https://github.com/omar0ali/portable-cloud-assignment1/blob/main/.github/workflows/main.yml) of the assignment 1 I worked on, since its already has the code to build and push the images to Amazon ECR. I copied the part that when it builds the image and push these images.
##### GitHub Action - Build and push images
###### Ensure the following setup in the GitHub Actions Secrets
2. `AWS_ACCESS_KEY_ID`
3. `AWS_SECRET_ACCESS_KEY`
4. `AWS_SESSION_TOKEN`

>[!NOTE]
The containerized application will use **pod**, **replicaset**, **deployment** and **service** manifests. Will expose web application using Service of type **NodePort**. And MySQL will be exposed using a Service of type **ClusterIP**. Lastly, a modification of the config file and another deployment to show a new version of the application.

**TODO**: Will need to verify if its possible.
Will be testing the local environment similarly as we did for [assignment1](https://github.com/omar0ali/portable-cloud-assignment1/tree/main?tab=readme-ov-file#testing-in-local-environment) The only difference is that in KIND will write deployment yaml files that will create multiple pods with specific variables or name.

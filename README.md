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

#### Step 1: ECR Login EC2 Instance
##### First `aws` credentials
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
kubectl create secret docker-registry ecr-secret --docker-server=<aws-account-id>.dkr.ecr.us-east-1.amazonaws.com --docker-username=AWS --docker-password="$(cat ecr-pass.txt)"
```

#### Step 2: Deploying Pods & Services

Firstly, will need to create a cluster node and **we need to ensure the port we are exposing is also mentioned in that configuration file.** 
###### Example

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30000
        hostPort: 30000
        listenAddress: "0.0.0.0"
        protocol: TCP
```

>[!IMPORTANT]
>As I was testing my deployments with a default imperative creation of the first cluster, ports weren't mapped to the host, so it took quite a long time until understanding what was missing. Once that was set, I recreated the cluster with that configuration file, and ensured that my pods were accessed by the host machine. Not to mention the creation of services for each type of pod, such as mysql (ClusterIP) and the app (NodePort) is important.
##### Started With
At the beginning I just wanted to have a working website, so I only used deployment kind for both mysql and web-application, and it worked after configuring the services as well, and used  the default namespaces. 
###### Splitting/Dividing the deployment into (pod, replicaset, deployment) 
I think this part seems redundant since we are repeating the same thing over three other configuration files.

One thing I had to keep in mind which is since each config file or manifest file are independent, thereby will create their own instances and that is not what I want, I only want them to be connected and share or match their labels through all the configurations, meaning if I applied the pod.yaml `kubectl apply -f app-pod.yaml` and then I used the replicaset.yaml `kubectl apply -f app-replicaset.yaml` they should know that there is one pod already created and based on the specified ***replicas*** it should ensure they don't exceed that set, because they all share the same labels.



>[!NOTE]
>`kubectl apply -f <file>.yaml`

>[!IMPORTANT]
>Here is a command that I used the most while debugging and encountering issues when running the pods of the flask application.
>`kubectl logs -l app=app --all-containers` (app) is the name of the pods. It showed errors and made it easy to fix problems quick.
>
>After every error I encounter I made sure to remove currently deployment and start over. I know I can update and apply again, but this is done for practice reason and learning. `kubectl delete deployment app-deployment` similar for the service `kubectl delete service app-service`


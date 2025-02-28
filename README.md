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
At the beginning I just wanted to have a working website, so I only used deployment KIND for both mysql and web-application, and it worked after configuring the services as well, and used  the default namespaces.
###### Splitting/Dividing the deployment into (pod, replicaset, deployment) 
I think this part seems redundant since we are repeating the same thing over three other configuration files. Since a deployment manifest seems sufficiently enough.

I took a look at [labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) to see how its used and what benefit do we get from using it. 

>Labels are intended to be used to specify identifying attributes of objects that are meaningful and relevant to users, but do not directly imply semantics --[Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

From that I understand there is no dependency between running pods, replicas_sets, and deployments after each other so they can use the pod that were initially created from i.e replica_set or pod. So its only used to organize and to select subsets of objects (such as pods).

Because I tried to do that and see what happens when I run (pod.yaml, replicaset.yaml and deployment.yaml) in order and check how many pods were created from that, and I saw, two pods, one from the pod.yaml and the second from the deployment. But initially we only need one pod of `mysql`. Thus, this confirms that there isn't any semantic or meaningful relationship between them.

```bash
[]$ kubectl get deployments
NAME               READY   UP-TO-DATE   AVAILABLE   AGE
mysql-deployment   1/1     1            1           7s
[]$ kubectl get replicaset
NAME                         DESIRED   CURRENT   READY   AGE
mysql-deployment-85d884657   1         1         1       22s
mysql-replicaset             1         1         1       16m
[]$ kubectl get pods
NAME                               READY   STATUS    RESTARTS   AGE
mysql                              1/1     Running   0          17m
mysql-deployment-85d884657-49c2r   1/1     Running   0          27s
[]$

```

What I see here that the deployment creates its own replica_set and pods, disregarding the fact there exists a replicas_set and a pod using the selector and `matchLabels`. So I think only the replica_set can manage the existing pods using the selector. 

I found an issue [github:#66742](https://github.com/kubernetes/kubernetes/issues/66742) someone suggested that because a replica_set was manually created first, that’s why the deployment did not take over. However, the real reason is that deployments do not adopt pre-existing replica_sets or pods, they always create their own. [reference](https://github.com/kubernetes/kubernetes/issues/66742#issuecomment-2054124694)

##### Continue
I believe that I will face the same issue when I start the employee flask application, and that it will create even more replicas, original a single pod from the first pod manifest file, then 3 replicas pods from the replica_set manifest file and another 3 from the deployment. 





>[!NOTE]
>`kubectl apply -f <file>.yaml`

>[!IMPORTANT]
>Here is a command that I used the most while debugging and encountering issues when running the pods of the flask application.
>`kubectl logs -l app=app --all-containers` (app) is the name of the pods. It showed errors and made it easy to fix problems quick.
>
>After every error I encounter I made sure to remove currently deployment and start over. I know I can update and apply again, but this is done for practice reason and learning. `kubectl delete deployment app-deployment` similar for the service `kubectl delete service app-service`


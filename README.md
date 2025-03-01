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
Referring to [main.yaml](https://github.com/omar0ali/clo835-k8s-cluster-assignment2/blob/main/.github/workflows/main.yaml), since we already used it in assignment 1 to build and push the images (mysql, flask_app) to Amazon ECR. I copied the part that when it builds the image and push them to amazon ECR.
##### GitHub Action - Build and push images
###### Ensure the following setup in the GitHub Actions Secrets
2. `AWS_ACCESS_KEY_ID`
3. `AWS_SECRET_ACCESS_KEY`
4. `AWS_SESSION_TOKEN`

>[!NOTE]
The containerized application will use **pod**, **replicaset**, **deployment** and **service** manifests. Will expose web application using Service of type **NodePort**. And MySQL will be exposed using a Service of type **ClusterIP**. Lastly, a modification of the config file and another deployment to show a new version of the application.

##### KIND & Kubectl Setup
To install kind here the [install documentation](https://kind.sigs.k8s.io/docs/user/quick-start/#installing-from-release-binaries) for linux

```bash
# For AMD64 / x86_64
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-amd64
# For ARM64
[ $(uname -m) = aarch64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.27.0/kind-linux-arm64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

Next will need `kubectl` installed as well. [install documentation](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/#install-kubectl-binary-with-curl-on-linux) 

```bash
 curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
```

```bash
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
```

Testing to ensure is installed

```bash
kubectl version --client
```

#### ECR Login EC2 Instance

>[!NOTE]
>I will be using the terms 'manifest' and 'configuration' interchangeably to refer to the descriptor files.
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

#### Deploying Pods & Services

>[!IMPORTANT]
>As I was testing my deployments with a default imperative way, the creation of the first cluster, ports weren't mapped to the host, so it took quite long time until understanding what was missing. Once that was set, I recreated the cluster with that configuration file, and ensured that my pods were accessed by the host machine. Not to mention the creation of services for each type of pod, such as mysql (ClusterIP) and the app (NodePort) is important.

Firstly, will need to create a cluster node and **we need to ensure the port we are exposing, that is also assigned in that configuration file.** 
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

To create the cluster first using the following manifest file.

```bash
kind create cluster --config  assignment2-cluster.yaml
```

This will ensure the cluster node forward traffic to the host. Will also ensure that ec2 instance port 30000 is open from using the security group.

##### Started With

At the beginning I just wanted to have a working website, so I only used deployment kind for both mysql and web-application, and it worked after configuring the services as well, and used the default namespaces.
###### Splitting/Dividing the deployment into (pod, replicaset, deployment) 

I think this part seems redundant since we are repeating the same thing over three other configuration files. Since a deployment manifest seems sufficiently enough.

I took a look at [labels and selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/) to see how its used and what benefit do we get from using it. 

>Labels are intended to be used to specify identifying attributes of objects that are meaningful and relevant to users, but do not directly imply semantics --[Labels and Selectors](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/)

From that I understand there is no dependency between running pods, replicas_sets, and deployments after each other so they can use the pod that were initially created from i.e ReplicaSet or pod. So its only used to organize and to select subsets of objects (such as pods).

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

What I see here that the deployment creates its own ReplicaSet and pods, disregarding the fact there exists a replicas_set and a pod using the selector and `matchLabels`. So I think only the replica_set can manage the existing pods using the selector. 

I found an issue [github:#66742](https://github.com/kubernetes/kubernetes/issues/66742) someone suggested that because a ReplicaSet was manually created first, that’s why the deployment did not take over. However, the real reason is that deployments do not adopt pre-existing ReplicaSets or pods, they always create their own. [reference](https://github.com/kubernetes/kubernetes/issues/66742#issuecomment-2054124694)

##### Continue

I believe that I will face the same issue when I start the employee flask application, and that it will create even more replicas, original a single pod from the first pod manifest file, then 3 replicas pods from the ReplicaSet manifest file and another 3 from the deployment. 

##### Namespaces

We will need a namespace for both (mysql and web application). 

```bash
kubectl create namespace mysql-space
kubectl create namespace employees-space
```

There are two directories that contains the k8s configs/manifest files that will deploy either pods, replicas, or deployments. First directory contains web application configs, and the second contains mysql configs.

As well there is the cluster configuration.

```bash
drwxr-xr-x. 4 ec2-user ec2-user    96 Feb 27 21:23 .
drwx------. 9 ec2-user ec2-user 16384 Feb 28 09:56 ..
-rw-r--r--. 1 ec2-user ec2-user   211 Feb 27 19:33 app-cluster.yaml
drwxr-xr-x. 2 ec2-user ec2-user    57 Feb 28 06:25 app-deployment
-rw-r--r--. 1 ec2-user ec2-user  2125 Feb 28 01:20 ecr-pass.txt
drwxr-xr-x. 2 ec2-user ec2-user   112 Feb 28 09:33 mysql-deployment
```

Will also contain the `ecr-pass.txt` which will be used to create a secret for kubectl to download the images that were assigned within manifest files.

>[!IMPORTANT]
>Here is a command that I used the most while debugging and encountering issues when running the pods of the flask application.
>`kubectl logs -l app=app --all-containers` (app) is the name of the pods. It showed errors and made it easy to fix problems quick.
>
>After every error I encounter I made sure to remove currently deployment and start over. I know I can update and apply again, but this is done for practice reason and learning. `kubectl delete deployment app-deployment` similar for the service `kubectl delete service app-service`

To connect to the Web App Pod

```bash
kubectl exec -it employees-pod -n employees-space -- sh
```

To test connection to MySQL

```bash
kubectl run testpod --rm -it --image=busybox:1.28 --restart=Never -n mysql-space -- nc -zv mysql-service.mysql-space.svc.cluster.local 3306
```

Check logs

```bash
kubectl logs demployees-pod -n employees-space
```

### Questions
###### What is the IP of the K8s API server in your cluster? (Answer in the report)

I can get the ip address by executing either of the following commands

```bash
docker ps
kubectl cluster-info
```

>[!NOTE]
>If we have multiple clusters, we will need to select the cluster using the the command i.e `kubectl config use-context kind-kind` [docs](https://kubernetes.io/docs/reference/kubectl/generated/kubectl_config/kubectl_config_set-context/)

![](School/CAA/Term%202/Portable%20Tech%20in%20Cloud/assignments/assignment2/screenshots/Pasted%20image%2020250228205604.png)

###### After deploying mysql and web applications pods with their respective namespaces. Can both applications listen on the same port inside the container?
Yes, both applications can listen on the same port inside their respective containers. Each pod has a unique IP address, even if they are in different namespaces.

The conflict happens **only if multiple services are exposed using the same NodePort** on a cluster, but **containerPorts inside pods do not conflict**.

Note that **MySQL Pod** using the namespace: `mysql-space` and listens on **3306**, and web App Pod using namespace: `employees-space` Listens on **8080**. Both can use the same ports inside their respective containers without issues.
###### Connect to the server running in the application pod and get a valid response.

![](screenshots/Pasted%20image%2020250301001602.png)

![](screenshots/Pasted%20image%2020250301001816.png)

This only was available once as well started the services for both mysql and the web application since they use the ports to ensure its accessibility. 
###### Examine the logs of the invoked application to demonstrate the response from the server was reflected in the log file 
![](screenshots/Pasted%20image%2020250301002714.png)
###### Explain. Use the “app:mysql” label to create ReplicaSets for MySQL application.
What I understood that in terms of taking ownership of the running pods using selector/labels applies here. There are other rules that were mentioned somewhere in the documentation `pod-template-hash`, but in this specific scenario, when `kubectl apply -f mysql-replicaset.yaml` that created a replicaset of mysql and took ownership of that running pod, so this replicaset now governs that pod. 
###### Use the labels from step 3 as selectors in the deployment manifest.
At this point where I saw the odd behavior, deployment didn't take ownership here, but it created its own replicaset and pods.  That is only applies on mysql. However, for the web application, that was not the case as I also have shown in the video, the deployment took govern over the replicaset that was created that was also governs the pod. 

>[!NOTE]
>This is the part where I lack knowledge. I observed an odd behavior when using kind: Deployment, where the MySQL Deployment did not take ownership of the ReplicaSet, but the Employees application Deployment did. I need to research further to understand why this happened.
###### Is the replicaset created in step 3 part of this deployment? Explain. 
For the web application (employees) yes, once the deployment was applied, I checked and saw that the replicaSet was taking over by the Deployment. 
###### Explain the reason we are using different service types for the web and MySQL applications. 
First, we have the cluster node, where we need to open a port for communication. Then, we have services, which I see as components that manage traffic between applications based on specific instructions.

For MySQL service, it is set to ClusterIP, meaning it only allows internal communication within the cluster. It ensures that traffic reaches the MySQL pod but does not expose it outside the cluster.

On the other hand, the web application service uses NodePort, which allows external access. This means it forwards traffic from outside the cluster to the appropriate application pod. That will give us access from outside of the cluster i.e using public ip address to access the web application.
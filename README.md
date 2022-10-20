# Create EKS cluster

<br>

## Steps to create EKS cluster

![Image](/images/eks-steps.png)

<br>

### 1 - Create IAM Role

this Role will be assigned to the EKS cluster to allow AWS to create and manage components on our behalf (allows access to other AWS service resources that are required to operate clusters managed by EKS).

### 2- Create a VPC for eks worker nodes
Why do we need a VPC? 
- EKS needs specfic networking configuration; it's based on k8s
- AWS has specific networking rules that are differents for k8s
So Default VPC is not optimized for eks, at least for the best practices for running an EKS cluster. 

<br>

![Image](/images/vpc-for-worker.png)

We can use [CloudFormation](https://docs.aws.amazon.com/eks/latest/userguide/creating-a-vpc.html) to create the VPC with all required configurations. 
This will create the stack that we need to create our cluster. 

<br>

![Image](/images/vpc-stack.png)

### 3- create EKS cluster 

After creating EKS cluster, we can connect to it locally with `kubectl`. 

```
aws eks update-kubeconfig --name eks-cluster-test --region eu-east-1
```
This command will create a config file wich have all the required configuration for `kubectl` to connect to the cluster
<br>

![Image](/images/kube-config.png)

<br>

### 4- Create a Node Group and attach it to EKS cluster to have all master and worker nodes running 
<br>
![Image](/images/master-worker.png)

<br>

- create `EC2 IAM Role` for Node group 
Worker nodes run worker processes and `Kubelet`(main process that schedule and manage pods, communicate with other AWS services) is one of them. So it needs some permissions to call aws API. 
We need 3 main permissions : 
- `AmazonEKSWorkerNodePolicy` : allows Amazon EKS worker nodes to connect to Amazon EKS Clusters.
- `AmazonEC2COntainerRegistryRead` : provides read-only access to Amazon EC2 Container Registry repositories.
- `AmazonEKS_CNI_Policy` : provides the Amazon VPC CNI Plugin (amazon-vpc-cni-k8s) the permissions it requires to modify the IP address configuration. 

<br>

![Image](/images/woker-node.png)

<br>

### 5- deploy our application to EKS cluster

Let's deploy nginx pods with servicex 

```
kubectl apply -f nginx.yaml
```

![Image](/images/nginx-deploy.png)

<br>

![Image](/images/nginx.png)


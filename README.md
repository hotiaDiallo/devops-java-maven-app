# Deploy to EKS and Dockerhub

### Environment 
We use Jenkins as docker container running on Digital Ocean Droplet

### Steps to deploy Image from Dockerhub to EKS

- Install [kubectl](https://kubernetes.io/fr/docs/tasks/tools/install-kubectl/) inside Jenkins container 
- Install [aws-iam-authenticator](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html) inside Jenkins container. 
Amazon EKS uses IAM to provide authentication to your Kubernetes cluster through the AWS IAM authenticator for Kubernetes
- reate a kubeconfig file for the cluster : The kubectl command-line tool uses configuration information in kubeconfig files to communicate with the API server of a cluster

```
aws eks update-kubeconfig --region region-code --name my-cluster
```

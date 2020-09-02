# KopsKubernetes
Guide to setup a kubernetes cluster in AWS using Kops

## Before you begin ##
You must have kubectl installed.

You must install kops on a 64-bit (AMD64 and Intel 64) device architecture.

You must have an AWS account, generate IAM keys and configure them. The IAM user will need adequate permissions.

## Installing kops on linux: ##

  ```
  curl -Lo kops https://github.com/kubernetes/kops/releases/download/$(curl -s https://api.github.com/repos/kubernetes/kops/releases/latest | grep tag_name | cut -d '"' -f 4)/kops-linux-amd64 
  
  chmod +x ./kops
  
  sudo mv ./kops /usr/local/bin/ 
  ``` 
 ## Installing Kubectl on linux: ##

``` 
curl -Lo kubectl https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
```
You can also refere the guide to install the components in different OS in the following link https://kubernetes.io/docs/setup/production-environment/tools/kops/ 



## Setup AWSCLI ##

Refer the guide https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html to install the aws-cli tool.

Once you've installed the AWS CLI tools and have correctly setup your system to use the official AWS methods of registering security credentials as defined here we'll be ready to run kops, as it uses the Go AWS SDK.


## Setup IAM user ##
In order to build clusters within AWS we'll create a dedicated IAM user for kops. This user requires API credentials in order to use kops. Create the user, and credentials, using the AWS console.

The kops user will require the following IAM permissions to function properly:

```
AmazonEC2FullAccess
AmazonRoute53FullAccess
AmazonS3FullAccess
IAMFullAccess
AmazonVPCFullAccess
AmazonEC2ContainerRegistryFullAccess
```
You should record the SecretAccessKey and AccessKeyID for the new IAM user.


## configure the aws client to use your new IAM user ##
```
aws configure           # Use your new access and secret key here
aws iam list-users      # you should see a list of all your IAM users here
```

## Because "aws configure" doesn't export these vars for kops to use, we export them now ##
```
export AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
export AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)
```

## Cluster State storage ##
In order to store the state of your cluster, and the representation of your cluster, we need to create a dedicated S3 bucket for kops to use. This bucket will become the source of truth for our cluster configuration. In this guide we'll call this bucket ```cluster-state-store```, but you should add a custom prefix as bucket names need to be unique.

We recommend keeping the creation of this bucket confined to us-east-1, otherwise more work will be required.

```
aws s3api create-bucket \
    --bucket cluster-state-store \
    --region us-east-1
```

## Creating your first cluster ## 

For a gossip-based cluster, make sure the name ends with k8s.local. For example:
```
export NAME=cluster.k8s.local
export KOPS_STATE_STORE=s3://cluster-state-store
```
Note: You don’t have to use environmental variables here. You can always define the values using the –name and –state flags later.

## Create cluster configuration ##

Below is a create cluster command. We'll use the most basic example possible, with more verbose examples in high availability. The below command will generate a cluster configuration, but will not start building it. Make sure you have generated an SSH key pair before creating your cluster.

```
kops create cluster \
    --zones=us-west-2a \
    ${NAME}
```

All instances created by kops will be built within ASG (Auto Scaling Groups), which means each instance will be automatically monitored and rebuilt by AWS if it suffers any failure.

## Create the actual cluster ##
```
kops create secret --name ${NAME} sshpublickey admin -i id_rsa.pub
kops update cluster ${NAME} --state=${KOPS_STATE_STORE} --yes
kops rolling-update cluster
```

## Validate cluster ##
```
kops validate cluster --wait 10m
```

## Use the Cluster ##
Remember when you installed kubectl earlier? The configuration for your cluster was automatically generated and written to ~/.kube/config for you!

A simple Kubernetes API call can be used to check if the API is online and listening. Let's use kubectl to check the nodes.

```
kubectl get nodes
```

You can look at all system components with the following command.
```
kubectl -n kube-system get po
```




### Delete the cluster ##
```
kops delete cluster --name ${NAME} --yes
```













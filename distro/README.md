# Set up a distributed environment for ECS plugin testing

This setup walks through building a distributed installation of Spinnaker on Amazon EKS for production level scale testing of ECS features. The Amazon EKS setup pieces are largely barrowed from [this blog post](https://aws.amazon.com/blogs/opensource/continuous-delivery-spinnaker-amazon-eks/) about using Spinnaker on Amazon EKS.

Useful applications of such a setup include:
* testing pipeline deployment UX/performance in accounts with large numbers of resources
* observing cache behavior (via API calls & error rates) in accounts with large numbers of resources
* testing effectiveness of performance enhancements

High level steps are:

 - [Create IAM resources](#create-iam-resources)
 - [Configure Cloud9 Halyard environment](#configure-cloud9-halyard-environment)
 - [Create Amazon EKS resources](#create-amazon-eks-resources)
 - [Configure and Launch Spinnaker]()
 - [Create Amazon ECS resources]()
 - [Set up AWS CloudTrail monitoring]()
 - [Create ECS pipelines]()
 - [Debugging and Observability]()

### Prerequisites
* AWS account 
* Permissions to [LOTS OF] resources.
* A Github account to retrieve Spinnaker artifacts from


## Create IAM resources

If you haven't already created the roles in [spinnaker-roles.yml](../spinnaker-roles.yml), do so:
```
aws cloudformation deploy --template-file spinnaker-roles.yml --region us-west-2 --stack-name SpinnakerRoles --capabilities CAPABILITY_NAMED_IAM
```

Create a node instance policy with the permissions clouddriver needs to run the AWS and ECS providers:
```
[CMD & TEMPLATE TBD]
```


## Configure Cloud9 Halyard environment

Open the AWS Cloud9 console:

https://us-west-2.console.aws.amazon.com/cloud9/home/product?region=us-west-2# 

Click on "Create environment". Give it the name "spinnaker-distro-admin". Select "m5.large" as your instance type, and "**Ubuntu Server**" as the platform. Adjust the hibernation time as desired, then complete the wizard with given values.

When Cloud9 environment launches, open a new terminal and install `kubectl` and `aws-iam-authenticator`:
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl

curl -o aws-iam-authenticator https://amazon-eks.s3-us-west-2.amazonaws.com/1.13.7/2019-06-11/bin/linux/amd64/aws-iam-authenticator

chmod +x ./aws-iam-authenticator

mkdir -p $HOME/bin && cp ./aws-iam-authenticator $HOME/bin/aws-iam-authenticator && export PATH=$HOME/bin:$PATH

echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc

aws-iam-authenticator  help   // should output help text to confirm install
```

Upgrade the `awscli` installed on the box:
```
pip install awscli --upgrade --user
```

Install [`eksctl`](https://eksctl.io/)
```
curl --silent --location "https://github.com/weaveworks/eksctl/releases/download/latest_release/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp

sudo mv /tmp/eksctl /usr/local/bin
```

[install terraform?]

Install halyard
```
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh

sudo bash InstallHalyard.sh   // when prompted to supply a non-root user, enter "ubuntu"

sudo update-halyard

hal -v
```

[configure role for Halyard env?]

## Create Amazon EKS Resources

Use `eksctl` to create a cluster nammed "eks-spinnaker" with 5 (to start - you may scale up or down later depending on your test needs) **m5.4xlarge** nodes (this is the instance type [used by Netflix to run clouddriver](https://medium.com/@rizza/scaling-clouddriver-at-netflix-b9ad7fc8b809)).

This step will take about 15 minutes to complete.
```
eksctl create cluster --name=eks-spinnaker --nodes=5 --ssh-public-key [KEY_PAIR_NAME] --node-type m5.4xlarge --write-kubeconfig=false --region=[SPINNAKER_REGION]
```

When cluster creation completes, confirm nodes are available:
```
kubectl get nodes
```

Retrieve the cluster and kubectl contexts:
```
aws eks update-kubeconfig --name eks-spinnaker --region=[SPINNAKER_REGION] --alias eks-spinnaker
```

## Configure and launch Spinnaker

TBD

## Create ECS resources

TBD
* networking and load balancer resources

## Set up AWS CloudTrail monitoring

TBD - will be stolen from: https://github.com/allisaurus/observe-spinnaker-aws


## Create ECS pipelines

* pipeline templates for ECS (if diff from higher up in this repo)
* task def to use
* CFN template for ECS

## Debugging and Observability

* how to find clouddriver and other Spinnaker service logs... in Spinnaker
* queries to run to observe cache/error behavior
* common issues
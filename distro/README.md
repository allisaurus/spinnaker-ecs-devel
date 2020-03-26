# Set up a distributed environment for ECS plugin testing

This setup walks through building a distributed installation of Spinnaker hosted on Amazon EKS for production-like scale testing of the Amazon ECS provider. 
The Amazon EKS setup pieces are largely barrowed from [this blog](https://aws.amazon.com/blogs/opensource/continuous-delivery-spinnaker-amazon-eks/) and [this workshop](https://eksworkshop.com/020_prerequisites/workspace/).

High level steps are:
 - [Create IAM resources](#create-iam-resources)
 - [Configure Cloud9 Halyard environment](#configure-cloud9-halyard-environment)
 - [Create Amazon EKS resources](#create-amazon-eks-resources)
 - [Configure and Launch Spinnaker](#configure-and-launch-spinnaker)
 - [Create Amazon ECS resources](#create-amazon-ecs-resources)
 - [Set up AWS CloudTrail monitoring](#set-up-aws-cloudtrail-monitoring)

### Prerequisites
* AWS account 
* Near-admin level permissions to see and create resources in many AWS services.
  * Including, but not limited to, IAM, EC2, ELB, EKS, S3, ECS and ECR.
* Any artifacts or technologies you need to deploy with Spinnaker.
  * E.g., github account, docker registries, etc.

## Create IAM resources

> NOTE: This section has gaps, specifically CloudFormation templates needed to generate some IAM roles and policies. 
> These are noted where applicable.

Create the following IAM Resources to be used by this setup:

1. `SpinDistroHalInstanceRole` : the role to be used by the Cloud9 Halyard environment to create and manage the Spinnaker installation and it's underlying infrastructure.
```
CMD & CFN Template TBD - good starting point: https://github.com/weaveworks/eksctl/issues/204
```

2. An access key for an IAM user with S3 permissions. This key will eventually be used by Front50 to manage Spinnaker's persistent storage.
  * Unfortunately, running Spinnaker on K8s (even on AWS) requires use of an access key vs. an IAM Role. See: https://www.spinnaker.io/setup/install/storage/s3/#prerequisites


3. The roles in [spinnaker-roles.yml](../spinnaker-roles.yml), if not already created:
```
aws cloudformation deploy --template-file ../spinnaker-roles.yml --region $AWS_REGION --stack-name SpinnakerRoles --capabilities CAPABILITY_NAMED_IAM
```

4. A `SpinnakerSupplementalNodePolicy` EC2 instance role policy which will allow Spinnaker nodes to access AWS:
```
[CMD & TEMPLATE TBD - should include lots of EC2 describes, assume-role permission for SpinnakerManaged role for AWS account in .hal/config]
```

## Configure Cloud9 Halyard environment

From the AWS Cloud9 console in your region of choice:

1. Click on "Create environment". 
2. Give it the name "spinnaker-distro-admin". 
3. Select "m5.large" as your instance type
4. Selct "**Ubuntu Server**" as the platform. 
5. Adjust the hibernation time and tags as desired.
6. Complete the wizard.

When Cloud9 environment launches, open a new terminal from within the IDE and install `kubectl` and [`aws-iam-authenticator`](https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html):
```
curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl

chmod +x ./kubectl

sudo mv ./kubectl /usr/local/bin/kubectl

curl -o aws-iam-authenticator https://amazon-eks.s3.us-west-2.amazonaws.com/1.15.10/2020-02-22/bin/linux/amd64/aws-iam-authenticator

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

eksctl version   // confirm it works

// enable bash completion
eksctl completion bash >> ~/.bash_completion
. /etc/profile.d/bash_completion.sh
. ~/.bash_completion
```

Install halyard
```
curl -O https://raw.githubusercontent.com/spinnaker/halyard/master/install/debian/InstallHalyard.sh

sudo bash InstallHalyard.sh   // when prompted to supply a non-root user, enter "ubuntu"

sudo update-halyard

hal -v
```

Install `jq`
```
sudo apt-get update -y
sudo apt-get install -y jq
```

Configure role for the Halyard env

1. Naviagate to the EC2 console in region where your Cloud9 instance is running, select the instance (name will start with "aws-cloud9-spin-distro-admin"), then choose 'Actions'/'Instance Settings'/'Attach/Replace IAM Role' and select previously created `SpinDistroHalInstanceRole`
2. In Cloud9 IDE, click the gear (settings) icon, go to 'AWS Settings'/'Credentials' and turn OFF 'AWS managed temporary credentials'.
3. Ensure temp creds aren't being used by removing existing file:
```
rm -vf ${HOME}/.aws/credentials
```
4. Set ACCOUNT_ID and AWS_REGION
```
export ACCOUNT_ID=$(aws sts get-caller-identity --output text --query Account)
export AWS_REGION=$(curl -s 169.254.169.254/latest/dynamic/instance-identity/document | jq -r '.region')
```
5. Save values to bash profile
```
echo "export ACCOUNT_ID=${ACCOUNT_ID}" | tee -a ~/.bash_profile
echo "export AWS_REGION=${AWS_REGION}" | tee -a ~/.bash_profile
aws configure set default.region ${AWS_REGION}
aws configure get default.region
```
6. Validate we're calling as the expected role
```
aws sts get-caller-identity --query Arn | grep SpinDistroHalInstanceRole -q && echo "IAM role valid" || echo "IAM role NOT valid"
```

## Create Amazon EKS Resources

Create and import a key pair we can use to SSH into EKS worker nodes from this instance.
```
ssh-keygen   // accept defaults

aws ec2 import-key-pair --key-name "SpinDistroKey" --public-key-material file://~/.ssh/id_rsa.pub
```

Use `eksctl` to create a cluster nammed "eks-spinnaker" with 5 (to start - you may scale up or down later depending on your test needs) **m5.4xlarge** nodes (this is the instance type [used by Netflix to run clouddriver](https://medium.com/@rizza/scaling-clouddriver-at-netflix-b9ad7fc8b809)).

This step will take about 15 minutes to complete.
* NOTE: `eksctl` will import `~/.ssh/id_rsa.pub` by default, so we don't need to pass it into this command.
```
eksctl create cluster --name=eks-spinnaker --nodes=5 --node-type=m5.4xlarge --region=${AWS_REGION}
```

When cluster creation completes, confirm nodes are available:
```
kubectl get nodes  // should output list of 5 nodes
```

## Configure and launch Spinnaker

1. Retrieve and alias the EKS cluster context:
```
aws eks update-kubeconfig --name eks-spinnaker --region=${AWS_REGION} --alias eks-spinnaker
```

2. Enable K8s with Halyard
```
hal config provider kubernetes enable
```

3. Configure a service account for the EKS cluster. (Learn more about [service accounts](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/))
```
kubectl config use-context eks-spinnaker

CONTEXT=$(kubectl config current-context)

// create a service account for the current context
kubectl apply --context $CONTEXT -f https://spinnaker.io/downloads/kubernetes/service-account.yml

// extract service account token
TOKEN=$(kubectl get secret --context $CONTEXT \
   $(kubectl get serviceaccount spinnaker-service-account \
       --context $CONTEXT \
       -n spinnaker \
       -o jsonpath='{.secrets[0].name}') \
   -n spinnaker \
   -o jsonpath='{.data.token}' | base64 --decode)

kubectl config set-credentials ${CONTEXT}-token-user --token $TOKEN

kubectl config set-context $CONTEXT --user ${CONTEXT}-token-user
```

4. Enable Kubernetes provider and add the `eks-spinnaker` account with Halyard
```
hal config provider kubernetes enable

hal config provider kubernetes account add eks-spinnaker --provider-version v2 --context $CONTEXT
```

5. Configure Spinnaker installation type, account, and S3 storage
 ```
hal config deploy edit --type distributed --account-name eks-spinnaker

// use previously generated S3 user access key; will prompt for matching secret key on execution
hal config storage s3 edit --access-key-id $ACCESS_KEY --secret-access-key --region $AWS_REGION

hal config storage edit --type s3
```

6. Set the desired Spinnaker version and deploy
```
// run 'hal version list' to see supported versions, run latest
hal config version edit --version $VERSION 

hal deploy apply
```

7. Verify the installation is working
```
kubectl -n spinnaker get svc   // should print names of running spinnaker services
```

8. Expose `deck` and `gate` services via ELB

> WARNING: This will expose Spinnaker to the internet! Make sure you complete the subsequent "lock down" steps or another auth mechanism immediately after this step.

```
export NAMESPACE=spinnaker

kubectl -n ${NAMESPACE} expose service spin-gate --type LoadBalancer \
  --port 80 --target-port 8084 --name spin-gate-public 

kubectl -n ${NAMESPACE} expose service spin-deck --type LoadBalancer \
  --port 80 --target-port 9000 --name spin-deck-public  

export API_URL=$(kubectl -n $NAMESPACE get svc spin-gate-public \
 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}') 

export UI_URL=$(kubectl -n $NAMESPACE get svc spin-deck-public \ 
 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}') 

hal config security api edit --override-base-url http://${API_URL} 

hal config security ui edit --override-base-url http://${UI_URL}

hal deploy apply
```

Reverify install, checking that `EXTERNAL-IP` values are set for `gate` and `deck`
```
kubectl -n spinnaker get svc
```

9. Test and lock down access to `deck` & `gate`
* Navigating to `deck` endpoint given by previous svc output
* Verfiy you can access Deck UI, view 'spin' application (and others)
* Lock down security groups for spin-deck-public and spin-gate-public to trusted IPs
* Lock down security group for spin-gate-public to trusted IPs
* Navigating to `deck` endpoint again, ensure you can view 'spin' application and clusters

## Create Amazon ECS resources

1. Configure AWS and ECS account with Halyard, per [these instructions](../README.md).
2. Attach the previously created `SpinnakerSupplementalNodePolicy` to the instance role for your Spinnaker nodes:
```
[CMDs TBD]
// get name of "NodeInstanceRole" from eksctl node group stack
// attach supplemental node policy to node instance role
```

## Set up AWS CloudTrail monitoring

In the region where Spinnaker is operating, follow [these instructions](https://github.com/allisaurus/observe-spinnaker-aws).

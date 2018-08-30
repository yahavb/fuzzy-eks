# fuzzy-eks
Deploy Helm-based packages using an EKS Cluster with a mix of EC2 OnDemand and EC2 Spot Instance types   

A method for setting up helm-based k8s packages deployed in an EKS Cluster with a mix of EC2 OnDemand and EC2 Spot Instance types. The process provided herein recommended for stateless workloads that require no durability or load-balancing like Massively Multiplayer Online game server. This method can be extended by using [traefik-ingress](https://github.com/pahud/amazon-eks-workshop/blob/master/03-creating-services/ingress/traefik-ingress/README.md)

We will go through the steps of deploying an EKS cluster from scratch, tools(`kubectl`,`aws-iam-authenticator`,`helm`, and `aws`), setup and deploy two types of worker node groups and finally deploy a helm package controlled by node affinity.

## Reference architecture
![alt_text](https://github.com/yahavb/fuzzy-eks/blob/master/images/arch-eks-helm.png)

## Setup Process
Like any AWS component, EKS utilizes IAM roles for allowing Kubernetes to perform AWS resources provisioning and deallocation. For example, you will need to create VPC, subnets, security groups, and load balancers that Kubernetes components will be deployed. Thus, it is required to create an IAM role that Kubernetes can assume to create AWS resources.
### [Create your Amazon EKS Service Role](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)

Make sure you use the two AWS managed policies: `AmazonEKSClusterPolicy` and `AmazonEKSServicePolicy`. 

### Create your Amazon EKS Cluster VPC

VPC and other EC2 components are provisioned thru CloudFormation templates. The EKS team maintain most of the templates we are going to use excluding the [EC2 Spot Instances template](https://github.com/yahavb/fuzzy-eks/blob/master/cloud-fomration/spot-nodegroup.yaml) Feel free to open an issue if you noticed the template needs an update. 
The EKS Cluster VPC template is available in the following [URL](https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2018-08-21/amazon-eks-vpc-sample.yaml). Please note that you will need to capture the `SecurityGroups`, `VpcId`, and `SubnetIds` for creating the cluster in the few steps. 

### Installing tools
* `kubectl` is the cmdline tool that allows interaction with the cluster API server. It is available in k8s [website](https://kubernetes.io/docs/tasks/tools/install-kubectl/) but also thru [Amazon EKS-vended](https://amazon-eks.s3-us-west-2.amazonaws.com/1.10.3/2018-07-26/bin/darwin/amd64/kubectl). 
* `aws-iam-authenticator` is used for authenticating the kubectl commands. The install instructions are available at [Amazon EKS-vended](https://docs.aws.amazon.com/eks/latest/userguide/configure-kubectl.html) 
* `awscli` is available at https://docs.aws.amazon.com/cli/latest/userguide/installing.html. 

### [Create Your Amazon EKS Cluster](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html)
Ideally, use the `awscli` for initiating the `aws eks create-cluster` as indicated. The reason is that the user you are going to use for creating the cluster will have to assume the role you created in the first step. In case of federate IAM identities, roles used thru the web-console will need to be reconfigured. 


### [Configure kubectl for Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html#Create%20Your%20Amazon%20EKS%20Cluster)
Follow the instructions listed in the kubectl configuration and make sure you are successfully able to execute `kubectl` commands against the cluster API server, e.g., `kubectl get svc`.



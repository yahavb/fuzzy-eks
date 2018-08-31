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

### [Launch and Configure Amazon EKS Worker Nodes](https://docs.aws.amazon.com/eks/latest/userguide/getting-started.html#Create%20Your%20Amazon%20EKS%20Cluster)
We will create two worker node groups, the first is On-Demand EC2 instances, the second is a diversified EC2 Spot instance. 
Both worker nodesgroups are provisioned using CloudFormation templates.
* [EC2 OD Instances template](https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2018-08-21/amazon-eks-nodegroup.yaml)
* [EC2 Spot Instances template](https://github.com/yahavb/fuzzy-eks/blob/master/cloud-fomration/spot-nodegroup.yaml)

For both templates, follow the instructions listed in ***Step 3: Launch and Configure Amazon EKS Worker Nodes***
When enabling ***worker nodes to join your cluster*** make sure this role has the policy `AmazonEKS_CNI_Policy` this policy enable the nodes in the node group to assign network resources from the Container Network Interface (CNI). Below is a list of Actions needed for the `NodeInstanceRole`
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:AssignPrivateIpAddresses",
                "ec2:AttachNetworkInterface",
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface",
                "ec2:DescribeInstances",
                "ec2:DescribeNetworkInterfaces",
                "ec2:DetachNetworkInterface",
                "ec2:ModifyNetworkInterfaceAttribute"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateTags"
            ],
            "Resource": [
                "arn:aws:ec2:*:*:network-interface/*"
            ]
        }
    ]
}
```
Once `aws-auth-cm.yaml` is applied, nodes can be joined the available pool. Feel free to skip ***Step 4: Launch a Guest Book Application*** as we are going to use Helm for deploying workloads. 

### Installing and Configure Helm
* Install Helm using [Helm project](https://github.com/helm/helm/releases)

Make sure that `helm version` returns healthy results

* Config Tiller 
Tiller is the Helm server-side component that requires k8s permissions via dedicated `serviceaccount`. Please note that no extra IAM roles are required for the Tiller to run as it uses a `serviceaccount`. The following command will create a service account for the tiller in `kube-system` namespace. 
```
kubectl create serviceaccount tiller --namespace kube-system
```
The next step is to bind the new serviceaccount `tiller` with the ClusterRole `cluster-admin`. You don't need to create `cluster-admin` role as it is a k8s predefined ClusterRole. 

```yaml
apiVersion: v1
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1beta1
metadata:
  name: tiller-role-binding
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
subjects:
- kind: ServiceAccount
  name: tiller
  namespace: kube-system
  ```

Apply [tiller-ClusterRoleBinding.yaml](https://github.com/yahavb/fuzzy-eks/blob/master/config/tiller-ClusterRoleBinding.yaml) by executing `kubectl apply -f tiller-ClusterRoleBinding.yaml` 

* Initialize the tiller
```
helm init --service-account tiller
```
Check the tiller is running in kube-system namespace
```
kubectl get pods --namespace kube-system | grep tiller
```

At this point Helm is ready for deploying packages. One can author its own, e.g., [nginx](https://github.com/yahavb/nginx-ananware) or search for public packages. 

* Update the helm repo
```
helm repo update
```
* Search for packages 
```
helm search| grep tomcat
```
```
helm install stable/tomcat
```
This will install the tomcat on your EKS cluster. 

### Taints/Tolerations and Spot/OD Node Labels
Spot instances provisioned by the nodegroup are labeled with the label `spotfleet=true`. We will use this label to control pod scheduling to available EC2 Spot Instances. Add the following section to the ***pod*** spec to indicate the scheduler to schedule the pod only to nodes that carry the label `spotfleet=true`. 

```yaml
     affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: spotfleet
                operator: In
                values:
                - "true"
```

### EC2 Spot Termination Handling
In case of instance termination, two states are possible, (1) the workload is not yet scheduled on a node to be terminated. (2) the workload is scheduled and required to be evicted before termination.   
#### Workload is not yet scheduled
Although Spot Instance interruption is becoming less unlikely event, its impact is perceived as a significant annoyance by users. [eks-lambda-drainer](https://github.com/pahud/eks-lambda-drainer) proposes a strategy to avoid these negative user impact. eks-lambda-drainer listen to spot termination signal from CloudWatch Events every 120 seconds in before the final termination process. Lambda function as the CloudWatch Event target, eks-lambda-drainer will perform the taint-based eviction on the terminating node. All pods without relative toleration will be evicted and rescheduled to another node. i.e., minimal impact on the spot instance termination.
#### Workload is running on a node that is going to be terminated
In the case of EC2 Spot instance interruption while the app is running, Kuberenetes supports Container lifecycle events. Kubetnetes supports postStart and preStop events. [nginx-spot.ymal](https://github.com/yahavb/fuzzy-eks/blob/master/specs/nginx-spot.yaml) shows an example of how to cofnigure PreStop event. Using PreStop event, the app can notify its users.

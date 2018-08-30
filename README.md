# fuzzy-eks
Deploy Helm-based packages using an EKS Cluster with a mix of EC2 OnDemand and EC2 Spot Instance types   

A method for setting up helm-based k8s packages deployed in an EKS Cluster with a mix of EC2 OnDemand and EC2 Spot Instance types. The process provided herein recommended for stateless workloads that require no durability or load-balancing like Massively Multiplayer Online game server. This method can be extended by using [traefik-ingress](https://github.com/pahud/amazon-eks-workshop/blob/master/03-creating-services/ingress/traefik-ingress/README.md)

We will go through the steps of deploying an EKS cluster from scratch, tools(`kubectl`,`aws-iam-authenticator`,`helm`, and `aws`), setup and deploy two types of worker node groups and finally deploy a helm package controlled by node affinity.

##Reference architecture
![alt_text]()


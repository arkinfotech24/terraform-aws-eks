# Infrastructure

![alt text](k8-infra.drawio.svg)

Creating above infrastructure involves lot of steps, as maintained sequence we need to create
* VPC
* All security groups and rules
* Bastion Host, VPN
* EKS
* RDS
* ACM for ingress
* ALB as ingress controller
* ECR repo to host images
* CDN

## Sequence

* (Required). create VPC first
* (Required). create SG after VPC
* (Required). create bastion host. It is used to connect RDS and EKS cluster.
* (Optional). VPN, same as bastion but a windows laptop can directly connect to VPN and get access of RDS and EKS.
* (Required). RDS. Create RDS because we don't create databases in Kubernetes.
* (Required). ACM. It is required to get SSL certificates for our ALB ingress controller.
* (Required). ingress ALB is required to expose our applications to outside world.
* (Required). ECR. We need to create ECR repo to host the application images.
* (Optional). CDN is optional. but good to have.

### Admin activities

**Bastion**
* SSH to bastion host
* run below command and configure the credentials.
```
aws configure
```
* get the kubernetes config using below command
```
aws eks update-kubeconfig --region us-east-1 --name <YOUR-CLUSTER-NAME>
```
* Now you should be able to connect K8 cluster
```
kubectl get nodes
```

**RDS**:
* Connect to RDS using bastion host.
```
mysql -h <DB-R53-ADRESS> -u root -pExpenseApp1
```
* We are creating schema while creating RDS. But table should be created.
* Refer backend.sql to create
    * Table
    * User
    * flush privileges

**Ingress Controller**

Ref: https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.8/
* Connect to K8 cluster from bastion host.
* Create an IAM OIDC provider. You can skip this step if you already have one for your cluster.
```
eksctl utils associate-iam-oidc-provider --region us-east-1 --cluster <your-cluster-name> --approve
```
* Download an IAM policy for the LBC using one of the following commands:
```
curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs aws-load-balancer-controller/v2.8.1/docs/install/iam_policy.json
```

* Create an IAM policy named AWSLoadBalancerControllerIAMPolicy. If you downloaded a different policy, replace iam-policy with the name of the policy that you downloaded.
```
aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy --policy-document file://iam-policy.json
```

* Create a IAM role and ServiceAccount for the AWS Load Balancer controller, use the ARN from the step above

```
eksctl create iamserviceaccount \
--cluster=<cluster-name> \
--namespace=kube-system \
--name=aws-load-balancer-controller \
--attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
--override-existing-serviceaccounts \
--approve
```

* Add the EKS chart repo to Helm
```
helm repo add eks https://aws.github.io/eks-charts
```

* Helm install command for clusters with IRSA:

```
helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=<cluster-name> --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
```

* check aws-load-balancer-controller is running in kube-system namespace.
```
kubectl get pods -n kube-system
```

```
𝗔𝗪𝗦 𝗜𝗻𝗳𝗿𝗮𝘀𝘁𝗿𝘂𝗰𝘁𝘂𝗿𝗲 𝗗𝗲𝗽𝗹𝗼𝘆𝗺𝗲𝗻𝘁 𝗳𝗼𝗿 𝗞𝘂𝗯𝗲𝗿𝗻𝗲𝘁𝗲𝘀 𝗔𝗽𝗽𝗹𝗶𝗰𝗮𝘁𝗶𝗼𝗻
Excited to share my part of my project where I successfully deployed a scalable application infrastructure on AWS using Terraform! 🚀
```
```
1. VPC Setup
Created a secure and scalable network environment with private and public subnets, route tables, NAT gateways, and internet gateways, all wrapped in a Terraform module.

2. Security Groups
Defined fine-grained access control for all resources including bastion, VPN, Load balancer, RDS, EKS cluster.

3. Bastion Host Setup
Provisioned a bastion EC2 instance in the public subnet to allow SSH access into the private network for troubleshooting and maintenance.

4. VPN Deployment
Configured a VPN EC2 instance to enable encrypted internal access for administrative operations.

5. EKS Cluster & Node Groups
Created a managed Amazon EKS Cluster with associated private worker nodes (Node Groups) for scalable container orchestration.

6. Ingress Controller
Set up the Ingress Controller in the public subnet to manage external HTTP/S traffic routed to the services running inside the EKS cluster.

7. Database
Deployed an RDS MySQL instance in the private subnet with encryption, backups, and restricted access.

8. Route53 & CDN
Configured Route 53 for DNS-based routing to the Ingress endpoint.
Integrated with a Content Delivery Network (CDN) to accelerate content delivery and reduce latency for users.
```


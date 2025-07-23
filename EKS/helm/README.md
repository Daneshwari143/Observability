# EKS Cluster Setup with eksctl and Robot Shop Microservices

This repository provides a step-by-step guide to creating an Amazon Elastic Kubernetes Service (EKS) cluster using `eksctl`, deploying the Robot Shop microservices application, and configuring the AWS Load Balancer Controller with an ingress resource. It includes instructions for setting up persistent storage with the AWS EBS CSI driver and troubleshooting stateful application issues.

## Features

- **EKS Cluster Creation**: Uses `eksctl` to provision an EKS cluster with managed nodes in the `ap-southeast-1` region.
- **Stateful Microservices**: Deploys the Robot Shop application with persistent storage support via the AWS EBS CSI driver for components like Redis.
- **Ingress Controller**: Configures the AWS Load Balancer Controller with IAM Roles for Service Accounts (IRSA) to manage Application Load Balancers (ALBs).
- **IAM OIDC Integration**: Sets up an IAM OIDC provider for secure authentication of Kubernetes service accounts.
- **Troubleshooting Guide**: Includes steps to resolve issues with stateful applications, such as PersistentVolumeClaim (PVC) binding errors.
- **Cleanup Instructions**: Provides commands to uninstall the application and controller for easy teardown.

## Prerequisites

To follow this guide, ensure you have the following:

- **AWS CLI**: Configured with a profile (e.g., `terraform-cloud-user`) and permissions for EKS, IAM, VPC, and EC2 in the `ap-southeast-1` region.
- **eksctl**: must be installed for cluster and addon management.
- **kubectl**: For interacting with the EKS cluster.
- **Helm**: For installing the Robot Shop application and AWS Load Balancer Controller.
- **AWS Account**: With permissions to create EKS clusters, IAM roles, policies, and VPC resources.
- **SSH Key (Optional)**: For Git authentication if cloning private repositories.

## Repository Structure

```
.
├── Chart.yaml
├── ingress.yaml
├── README.md
├── storage-class-gp3.yml
├── templates
│   ├── cart-deployment.yaml
│   ├── cart-service.yaml
│   ├── catalogue-deployment.yaml
│   ├── catalogue-service.yaml
│   ├── clusterrolebinding.yaml
│   ├── clusterrole.yaml
│   ├── dispatch-deployment.yaml
│   ├── dispatch-service.yaml
│   ├── mongodb-deployment.yaml
│   ├── mongodb-service.yaml
│   ├── mysql-deployment.yaml
│   ├── mysql-service.yaml
│   ├── payment-deployment.yaml
│   ├── payment-service.yaml
│   ├── podsecuritypolicy.yaml
│   ├── rabbitmq-deployment.yaml
│   ├── rabbitmq-service.yaml
│   ├── ratings-deployment.yaml
│   ├── ratings-service.yaml
│   ├── redis-service.yaml
│   ├── redis-statefulset.yaml
│   ├── serviceaccount.yaml
│   ├── shipping-deployment.yaml
│   ├── shipping-service.yaml
│   ├── user-deployment.yaml
│   ├── user-service.yaml
│   ├── web-deployment.yaml
│   └── web-service.yaml
└── values.yaml
```

*Note*: The Robot Shop Helm chart is assumed to be available in the repository’s root directory (`.`). If hosted elsewhere, update the Helm install command accordingly.

## Setup Instructions

### 1. Configure AWS Credentials
Ensure your AWS CLI is configured with the `<your-aws-profile>` profile:
```bash
aws configure --profile terraform-cloud-user
```
Set the region to `<your-aws-region>` and provide valid access/secret keys with permissions for EKS, IAM, VPC, and EC2.

### 2. Clone the Repository
```bash
git clone https://github.com/cloudhein/robot-shop.git

```

### 3. Create the EKS Cluster
Create an EKS cluster named `robot-shop-cluster` with two `t3.medium` nodes:
```bash
eksctl create cluster \
  --name robot-shop-cluster \
  --region ap-southeast-1 \
  --profile terraform-cloud-user \
  --nodes 2 \
  --node-type t3.medium \
  --nodegroup-name standard-workers
```

Verify the nodes:
```bash
kubectl get nodes
```
Expected output:
```
NAME                                                STATUS   ROLES    AGE   VERSION
ip-192-168-38-159.ap-southeast-1.compute.internal   Ready    <none>   36m   v1.32.3-eks-473151a
ip-192-168-70-10.ap-southeast-1.compute.internal    Ready    <none>   36m   v1.32.3-eks-473151a
```

### 4. Associate IAM OIDC Provider
Set up an IAM OIDC provider for the cluster to enable IRSA:
```bash
export cluster_name=robot-shop-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text --profile terraform-cloud-user --region ap-southeast-1 | cut -d '/' -f 5)
aws iam list-open-id-connect-providers --profile terraform-cloud-user | grep $oidc_id | cut -d "/" -f4
eksctl utils associate-iam-oidc-provider --cluster $cluster_name --profile terraform-cloud-user --region ap-southeast-1 --approve
```

### 5. Install AWS EBS CSI Driver
The EBS CSI driver enables persistent storage for stateful applications. Create the IAM role and service account:
```bash
eksctl create iamserviceaccount \
    --name ebs-csi-controller-sa \
    --namespace kube-system \
    --cluster robot-shop-cluster \
    --role-name AmazonEKS_EBS_CSI_DriverRole \
    --role-only \
    --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy \
    --profile terraform-cloud-user \
    --region ap-southeast-1 \
    --approve
```

Install the EBS CSI driver addon:
```bash
eksctl create addon \
  --name aws-ebs-csi-driver \
  --cluster robot-shop-cluster \
  --region ap-southeast-1 \
  --service-account-role-arn arn:aws:iam::730335247947:role/AmazonEKS_EBS_CSI_DriverRole \
  --profile terraform-cloud-user \
  --force
```
Reference: [AWS EBS CSI Driver Documentation](https://repost.aws/knowledge-center/eks-persistent-storage)

### 6. Deploy Robot Shop Microservices
Deploy the Robot Shop application using Helm in the `robot-ns` namespace:
```bash
helm install robot-shop . --create-namespace -n robot-ns
```

Verify the Helm release:
```bash
helm list -A
```
Expected output:
```
NAME            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                   APP VERSION
robot-shop      robot-ns        1               2025-07-21 17:47:10.067352661 +0700 +07 deployed        robot-shop-1.1.0
```

Check the deployed pods:
```bash
kubectl get pods -n robot-ns
```
Expected output (after resolving storage issues):
```
NAME                        READY   STATUS    RESTARTS   AGE
cart-655b74fb49-cqlk9       1/1     Running   0          36m
catalogue-b4855db44-snjtk   1/1     Running   0          36m
dispatch-845799dc84-zbxb9   1/1     Running   0          36m
mongodb-69d9cf5747-58wdq    1/1     Running   0          36m
mysql-8c599b989-kr29t       1/1     Running   0          36m
payment-6589fd67f6-sgr2q    1/1     Running   0          36m
rabbitmq-876447689-zwkqq    1/1     Running   0          36m
ratings-6fb5c59f44-8p2v6    1/1     Running   0          36m
redis-0                     1/1     Running   0          36m
shipping-67cdd8c8c6-5ljj6   1/1     Running   0          36m
user-b4977f556-cwnwf        1/1     Running   0          36m
web-7649bf4886-hxg2d        1/1     Running   0          36m
```

### 7. Install AWS Load Balancer Controller
Set up the IAM policy and service account for the AWS Load Balancer Controller:
```bash
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.13.3/docs/install/iam_policy.json
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json \
    --profile terraform-cloud-user
eksctl create iamserviceaccount \
    --cluster=robot-shop-cluster \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::730335247947:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --region ap-southeast-1 \
    --profile terraform-cloud-user \
    --approve
```

Install the controller using Helm:
```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=robot-shop-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set vpcId=vpc-028e7e406a9a1db45 \
  --version 1.13.0
```

Verify the Helm release:
```bash
helm list -A
```
Expected output:
```
NAME                            NAMESPACE       REVISION        UPDATED                                 STATUS          CHART                                   APP VERSION
aws-load-balancer-controller    kube-system     1               2025-07-21 17:57:43.027546001 +0700 +07 deployed        aws-load-balancer-controller-1.13.0     v2.13.0
robot-shop                      robot-ns        2               2025-07-21 17:51:43.208024434 +0700 +07 deployed        robot-shop-1.1.0
```

Reference: [AWS Load Balancer Controller Documentation](https://docs.aws.amazon.com/eks/latest/userguide/lbc-helm.html)

### 8. Deploy Ingress Resource
Apply the ingress resource to expose the Robot Shop application via an Application Load Balancer (ALB):
```bash
kubectl apply -f ingress.yaml
```

Verify the ingress:
```bash
kubectl get ingress -A
```
Expected output:
```
NAMESPACE   NAME         CLASS   HOSTS   ADDRESS                                                                       PORTS   AGE
robot-ns    robot-shop   alb     *       k8s-robotns-robotsho-bf6cb4cd48-2090334471.ap-southeast-1.elb.amazonaws.com   80      98s
```

Access the Robot Shop application using the ALB address (e.g., `http://k8s-robotns-robotsho-bf6cb4cd48-2090334471.ap-southeast-1.elb.amazonaws.com`).

### 9. Troubleshoot Stateful Applications
If the `redis-0` pod is stuck in `Pending` status due to unbound PersistentVolumeClaims (PVCs), follow these steps:

Check the pod status:
```bash
kubectl describe pod redis-0 -n robot-ns
```
Example output indicating a PVC issue:
```
Events:
  Type     Reason            Age                  From               Message
  ----     ------            ----                 ----               -------
  Warning  FailedScheduling  4m31s (x6 over 29m)  default-scheduler  0/2 nodes are available: pod has unbound immediate PersistentVolumeClaims.
```

Verify the PVC status:
```bash
kubectl get pvc -n robot-ns
```
Example output:
```
NAME           STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
data-redis-0   Pending                                      gp3            <unset>                 30m
```

Check available StorageClasses:
```bash
kubectl get storageclass
```
Example output:
```
NAME   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2    kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  94m
```

The `gp3` StorageClass is missing. Create it by applying the `storage-class-gp3.yml` file:
```bash
kubectl apply -f storage-class-gp3.yml
```
Example `storage-class-gp3.yml`:
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

Verify the StorageClass:
```bash
kubectl get sc
```
Expected output:
```
NAME   PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
gp2    kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  100m
gp3    kubernetes.io/aws-ebs   Delete          WaitForFirstConsumer   false                  5s
```

Recheck the pods to confirm all are running:
```bash
kubectl get pods -n robot-ns
```
Expected output:
```
NAME                        READY   STATUS    RESTARTS   AGE
cart-655b74fb49-cqlk9       1/1     Running   0          36m
catalogue-b4855db44-snjtk   1/1     Running   0          36m
dispatch-845799dc84-zbxb9   1/1     Running   0          36m
mongodb-69d9cf5747-58wdq    1/1     Running   0          36m
mysql-8c599b989-kr29t       1/1     Running   0          36m
payment-6589fd67f6-sgr2q    1/1     Running   0          36m
rabbitmq-876447689-zwkqq    1/1     Running   0          36m
ratings-6fb5c59f44-8p2v6    1/1     Running   0          36m
redis-0                     1/1     Running   0          36m
shipping-67cdd8c8c6-5ljj6   1/1     Running   0          36m
user-b4977f556-cwnwf        1/1     Running   0          36m
web-7649bf4886-hxg2d        1/1     Running   0          36m
```

### 10. Cleanup
To remove the deployed resources:
```bash
helm uninstall aws-load-balancer-controller -n kube-system
helm uninstall robot-shop -n robot-ns
```

To delete the EKS cluster:
```bash
eksctl delete cluster --name robot-shop-cluster --region ap-southeast-1 --profile terraform-cloud-user
```

## Usage Notes
- **Region**: The commands are configured for `ap-southeast-1`. Update the `--region` flag if using a different region.
- **Permissions**: Ensure the `terraform-cloud-user` profile has permissions for EKS, IAM, VPC, and EC2 resources.
- **VPC ID**: Replace `vpc-028e7e406a9a1db45` in the Helm install command with your actual VPC ID, obtainable from the AWS Console (VPC > Your VPCs).
- **IAM Role ARN**: Update `arn:aws:iam::730335247947:role/AmazonEKS_EBS_CSI_DriverRole` and `arn:aws:iam::730335247947:policy/AWSLoadBalancerControllerIAMPolicy` with your AWS account ID.
- **Ingress YAML**: Ensure the `ingress.yaml` file is available in your repository. A sample `ingress.yaml` might look like:
  ```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: robot-shop
  namespace: robot-ns
  labels:
    app: robot-shop
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  ingressClassName: alb
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 8080
  ```
- **StorageClass YAML**: The `storage-class-gp3.yml` file is included to resolve PVC issues for stateful applications like Redis.
- **Robot Shop Helm Chart**: The `helm install robot-shop .` command assumes the chart is in the repository’s root directory. If hosted elsewhere, update the command to point to the chart’s location.

## Contributing
Contributions are welcome! Please submit a pull request or open an issue for suggestions or bug reports.

## License
This project is licensed under the MIT License. See the [LICENSE](LICENSE) file for details.
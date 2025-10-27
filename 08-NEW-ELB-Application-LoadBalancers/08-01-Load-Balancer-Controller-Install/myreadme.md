1-check version
2-create eks cluster and worker nodes
# Create Cluster
eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup 

eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster eksdemo1 \
    --approve


# Create eks Node Group   in private subnets
eksctl create nodegroup --cluster=eksdemo1 \
                        --region=us-east-1 \
                        --name=eksdemo1-ng-public1 \
                        --node-type=t3.medium \
                        --nodes=2 \
                        --nodes-min=2 \
                        --nodes-max=4 \
                        --node-volume-size=20 \
                        --ssh-access \
                        --ssh-public-key=kube-demo \
                        --managed \
                        --asg-access \
                        --external-dns-access \
                        --full-ecr-access \
                        --appmesh-access \
                        --alb-ingress-access \
                        --node-private-networking

3-eksctl get cluster --region us-east-1

eksctl get nodegroup --cluster=eksdemo1 --region=us-east-1

4- create IAM policy
curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

# I-Create IAM Policy using policy downloaded 
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_latest.json

copy arn:aws:iam::502845302465:policy/AWSLoadBalancerControllerIAMPolicy

# II-Create an IAM role for the AWS LoadBalancer Controller and attach the role to the Kubernetes service account
# Verify if any existing service account
kubectl get sa -n kube-system
kubectl get sa aws-load-balancer-controller -n kube-system
kubectl describe sa aws-load-balancer-controller -n kube-system

Obseravation:
1. Nothing with name "aws-load-balancer-controller" should exist
# Replaced name, cluster and policy arn (Policy arn we took note in step-02)
eksctl create iamserviceaccount \
  --cluster=eksdemo1 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::502845302465:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --region us-east-1
  --approve

This command will 
 * create an AWS IAM role
 * create Kubernetes Service Account in k8s cluster
 * bound IAM Role created and the Kubernetes service account created

 eksctl get iamserviceaccount --cluster=eksdemo1 --region us-east-1
kubectl describe sa aws-load-balancer-controller -n kube-system
# Verify CloudFormation Template eksctl created & IAM Role

# copy the role: 
 arn:aws:iam::502845302465:role/eksctl-eksdemo1-addon-iamserviceaccount-kube--Role1-HCoHADkh4YNR

#  Install the AWS Load Balancer Controller using Helm V3
# Add the eks-charts repository.
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=eksdemo1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-0fe360a6165d2287c \
  --set image.repository=public.ecr.aws/eks/aws-load-balancer-controller

# Verify that the controller is installed and Webhook Service created
602401143452.dkr.ecr.us-east-1.amazonaws.com

# Create IngressClass Resource
kubectl apply -f kube-manifests
kubectl describe ingressclass my-aws-ingress-class


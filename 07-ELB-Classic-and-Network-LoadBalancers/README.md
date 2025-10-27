# AWS Elastic Load Balancers

## AWS Load Balancer Types
1. Classic Load Balancer
2. Network Load Balancer
3. Application Load Balancer  (k8s Ingress)

# step1: get nodegroup 
eksctl get nodegroup --cluster=eksdemo1 --region us-east-1

# step2: delete nodegroup in public subnets
eksctl delete nodegroup eksdemo1-ng-public1 --cluster eksdemo1 --region us-east-1

# step3: create EKS node gorup in private subnets
eksctl create nodegroup --cluster=eksdemo1 \
                        --region=us-east-1 \
                        --name=eksdemo1-ng-private1 \
                        --node-type=t3.medium \
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

kubectl get nodes -o wide
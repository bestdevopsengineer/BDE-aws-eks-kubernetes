## Step-01: Create EKS Cluster using eksctl
        eksctl create cluster --name=eksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup 
## Step-02: Create & Associate IAM OIDC Provider for our EKS Cluster
        To enable and use AWS IAM roles for Kubernetes service accounts on our EKS cluster
        eksctl utils associate-iam-oidc-provider \
            --region us-east-1 \
            --cluster eksdemo1 \
            --approve
## Step-03: Create EC2 Keypair
This will help us to login to the EKS Worker Nodes using Terminal.

## Step-04: Create Node Group with additional Add-Ons in Public Subnets
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
                        --alb-ingress-access 
![alt text](image.png)
![alt text](image-1.png)

Install the EKS Pod Identity Agent add-on
# Step-00: What we’ll do
# 1. Install the **EKS Pod Identity Agent** add-on  
        This installs a **DaemonSet** (`eks-pod-identity-agent`) that enables Pod Identity associations.
        kubectl get daemonset -n kube-system
        kubectl get pods -n kube-system
        * add on pod identity from cluster
        Create role for pod to read s3 bucket 
        name:
        EKS-PodIdentity-S3-ReadOnly-Role-101

        iam/role/aws service/eks/EKS - Pod Identity/s3/create

## Step-02-01: Create Service Account 
        01_k8s_service_account.yaml :
        apiVersion: v1
        kind: ServiceAccount
        metadata:
        name: aws-cli-sa
        namespace: default

## Create a simple Kubernetes Pod with AWS CLI image:
        02_k8s_aws_cli_pod.yaml :
        apiVersion: v1
        kind: Pod
        metadata:
        name: aws-cli
        namespace: default
        spec:
        serviceAccountName: aws-cli-sa
        containers:
        - name: aws-cli
            image: amazon/aws-cli
            command: ["sleep", "infinity"]

## goto cluster
        eksdemo1/access/pod identity/EKS-PodIdentity-S3-ReadOnly-Role-101/default/aws-cli-sa

        k apply -f .
        kubectl exec -it aws-cli -- aws s3 ls

![alt text](image-2.png)


# Install EBS CSI DRIVER
        goto eks/addon/ebs csi driver/eks pod identity/create role :AmazonEKSPodIdentityAmazonEBSCSIDriverRole1 
        eks/eks pod identity/
        AmazonEBSCSIDriverPolicy
        AmazonEKSClusterPolicy

# CREATE A STORAGE CLASS
        storage_class.yaml:  
            apiVersion: storage.k8s.io/v1
            kind: StorageClass
            metadata: 
            name: ebs-sc
            provisioner: ebs.csi.aws.com
            volumeBindingMode: WaitForFirstConsumer 
        
        persistent_volume_claim.yaml:
            apiVersion: v1
            kind: PersistentVolumeClaim
            metadata:
            name: ebs-mysql-pv-claim
            spec: 
            accessModes:
                - ReadWriteOnce
            storageClassName: ebs-sc
            resources: 
                requests:
                storage: 4Gi
        
        UserManagement_ConfigMap.yaml:
            apiVersion: v1
            kind: ConfigMap
            metadata:
            name: usermanagement-dbcreation-script
            data: 
            mysql_usermgmt.sql: |-
                DROP DATABASE IF EXISTS usermgmt;
                CREATE DATABASE usermgmt; 

        mysql_deployment.yaml:
            apiVersion: apps/v1
            kind: Deployment
            metadata:
            name: mysql
            spec: 
            replicas: 1
            selector:
                matchLabels:
                app: mysql
            strategy:
                type: Recreate 
            template: 
                metadata: 
                labels: 
                    app: mysql
                spec: 
                containers:
                    - name: mysql
                    image: mysql:5.6
                    env:
                        - name: MYSQL_ROOT_PASSWORD
                        value: dbpassword11
                    ports:
                        - containerPort: 3306
                        name: mysql    
                    volumeMounts:
                        - name: mysql-persistent-storage
                        mountPath: /var/lib/mysql    
                        - name: usermanagement-dbcreation-script
                        mountPath: /docker-entrypoint-initdb.d                 
                volumes: 
                    - name: mysql-persistent-storage
                    persistentVolumeClaim:
                        claimName: ebs-mysql-pv-claim
                    - name: usermanagement-dbcreation-script
                    configMap:
                        name: usermanagement-dbcreation-script
                
        mysql_clusterip_service.yaml:
            apiVersion: v1
            kind: Service
            metadata: 
            name: mysql
            spec:
            selector:
                app: mysql 
            ports: 
                - port: 3306  
            clusterIP: None 

    k get sc
    k get pvc
    

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
# #############################################################################################################
# this show that to list s3 bucket from eks/pod you need a service account
# #############################################################################################################
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
# #############################################################################################################
# this show that you need to have volume in pod to have a pv that pound to pvc
# #############################################################################################################

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

# CREATE A PVC
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

# CREATE A CONFIGMAP        
        UserManagement_ConfigMap.yaml:

            apiVersion: v1
            kind: ConfigMap
            metadata:
              name: usermanagement-dbcreation-script
            data: 
            mysql_usermgmt.sql: |-
              DROP DATABASE IF EXISTS usermgmt;
              CREATE DATABASE usermgmt; 

# CREATE A mysql deployment
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
                    claimName: ebs-mysql-pv-claim  # this create the pv and help to connect to pvc 
                - name: usermanagement-dbcreation-script
                  configMap:
                    name: usermanagement-dbcreation-script

# CREATE A mysql service               
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

# Connect to MySQL Database

        # Connect to MYSQL Database
        kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -pdbpassword11

        [or]

        # Use mysql client latest tag
        kubectl run -it --rm --image=mysql:latest --restart=Never mysql-client -- mysql -h mysql -pdbpassword11

        # Verify usermgmt schema got created which we provided in ConfigMap
        mysql> show schemas;

# create usermanagement deploy
     usermanagement_deploy.yaml:

        apiVersion: apps/v1
        kind: Deployment 
        metadata:
          name: usermgmt-microservice
          labels:
            app: usermgmt-restapp
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: usermgmt-restapp
          template:  
            metadata:
              labels: 
                app: usermgmt-restapp
            spec:
              containers:
                - name: usermgmt-restapp
                  image: stacksimplify/kube-usermanagement-microservice:1.0.0
                  ports: 
                    - containerPort: 8095           
                  env:
                    - name: DB_HOSTNAME
                    value: "mysql"            
                    - name: DB_PORT
                    value: "3306"            
                    - name: DB_NAME
                    value: "usermgmt"            
                    - name: DB_USERNAME
                    value: "root"            
                    - name: DB_PASSWORD
                    value: "dbpassword11"     

# create usermanagement service
        usermanagement-service.yaml:

        apiVersion: v1
        kind: Service
        metadata:
          name: usermgmt-restapp-service
          labels: 
            app: usermgmt-restapp
        spec:
          type: NodePort
        selector:
          app: usermgmt-restapp
        ports: 
          - port: 8095
            targetPort: 8095
            nodePort: 31231

        http://<EKS-WorkerNode-Public-IP>:31231/usermgmt/health-status
        add port in security nodes group 31231 -> 0.0.0.0/0
        http://50.19.8.112:31231/usermgmt/health-status

        you can test with postman by importing the file
        AWS-EKS-Masterclass-Microservices.postman_collection.json
        create/update/
        kubectl run -it --rm --image=mysql:5.6 --restart=Never mysql-client -- mysql -h mysql -u root -pdbpassword11
        mysql> show schemas;
        mysql> use usermgmt;
        mysql> show tables;
        mysql> select * from users;
# #############################################################################################################
# secrets/init containers/liveness-readiness probe/requests-limits/namespaces
# #############################################################################################################
![alt text](image-3.png)
        URL: https://www.base64encode.org
        echo -n 'dbpassword11' | base64

# Create Kubernetes Secrets manifest
    kubernetes_secrets.yaml:

        apiVersion: v1
        kind: Secret
        metadata:
          name: mysql-db-password
        type: Opaque
        data: 
          db-password: ZGJwYXNzd29yZDEx

# Update A mysql deployment with secret
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
                      valueFrom:
                        secretKeyRef:
                          name: mysql-db-password
                          key: db-password 
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
                    claimName: ebs-mysql-pv-claim  # this create the pv and help to connect to pvc 
                - name: usermanagement-dbcreation-script
                  configMap:
                    name: usermanagement-dbcreation-script

# update usermanagement deploy with secret
     usermanagement_deploy.yaml:

        apiVersion: apps/v1
        kind: Deployment 
        metadata:
          name: usermgmt-microservice
          labels:
            app: usermgmt-restapp
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: usermgmt-restapp
          template:  
            metadata:
              labels: 
                app: usermgmt-restapp
            spec:
              containers:
                - name: usermgmt-restapp
                  image: stacksimplify/kube-usermanagement-microservice:1.0.0
                  ports: 
                    - containerPort: 8095           
                  env:
                    - name: DB_HOSTNAME
                      value: "mysql"            
                    - name: DB_PORT
                      value: "3306"            
                    - name: DB_NAME
                      value: "usermgmt"            
                    - name: DB_USERNAME
                      value: "root"            
                    - name: DB_PASSWORD
                      valueFrom:
                        secretKeyRef:
                          name: mysql-db-password
                          key: db-password              

# use init container to check if mysql pod is up or not before connection
     usermanagement_deploy.yaml:

        apiVersion: apps/v1
        kind: Deployment 
        metadata:
          name: usermgmt-microservice
          labels:
            app: usermgmt-restapp
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: usermgmt-restapp
          template:  
            metadata:
              labels: 
                app: usermgmt-restapp
            spec:
              initContainers:
                - name: init-db
                  image: busybox:1.31
                  command: ['sh', '-c', 'echo -e "Checking for the availability of MySQL Server deployment"; while ! nc -z  mysql 3306; do sleep 1; printf "-"; done; echo -e "  >> MySQL DB Server has started";']
              containers:
                - name: usermgmt-restapp
                  image: stacksimplify/kube-usermanagement-microservice:1.0.0
                  ports: 
                    - containerPort: 8095           
                  env:
                    - name: DB_HOSTNAME
                      value: "mysql"            
                    - name: DB_PORT
                      value: "3306"            
                    - name: DB_NAME
                      value: "usermgmt"            
                    - name: DB_USERNAME
                      value: "root"            
                    - name: DB_PASSWORD
                      valueFrom:
                        secretKeyRef:
                          name: mysql-db-password
                          key: db-password            

# liveness probe
            **Observation:** User Management Microservice pod witll not be in READY state to accept traffic until it completes the `initialDelaySeconds=60seconds`

            when to restart a container

        usermanagement_deploy.yaml:

        apiVersion: apps/v1
        kind: Deployment 
        metadata:
          name: usermgmt-microservice
          labels:
            app: usermgmt-restapp
        spec:
          replicas: 1
          selector:
            matchLabels:
              app: usermgmt-restapp
          template:  
            metadata:
              labels: 
                app: usermgmt-restapp
            spec:
              initContainers:
                - name: init-db
                  image: busybox:1.31
                  command: ['sh', '-c', 'echo -e "Checking for the availability of MySQL Server deployment"; while ! nc -z  mysql 3306; do sleep 1; printf "-"; done; echo -e "  >> MySQL DB Server has started";']
              containers:
                - name: usermgmt-restapp
                  image: stacksimplify/kube-usermanagement-microservice:1.0.0
                  ports: 
                    - containerPort: 8095           
                  env:
                    - name: DB_HOSTNAME
                      value: "mysql"            
                    - name: DB_PORT
                      value: "3306"            
                    - name: DB_NAME
                      value: "usermgmt"            
                    - name: DB_USERNAME
                      value: "root"            
                    - name: DB_PASSWORD
                      valueFrom:
                        secretKeyRef:
                          name: mysql-db-password
                          key: db-password     
                  livenessProbe:
                    exec:
                      command: 
                        - /bin/sh
                        - -c 
                        - nc -z localhost 8095
                    initialDelaySeconds: 60
                    periodSeconds: 10
                  
                  readinessProbe:
                    httpGet:
                      path: /usermgmt/health-status
                      port: 8095
                    initialDelaySeconds: 60
                    periodSeconds: 10                   

# readiness probe
        when a container is ready to accept traffic
# startup probe
        when a container application has started
# options to define probes
        check using commands: /bin/sh -c nc -z localhost 8095
        check using HTTP GET Request: httpget path:/health-status
        check using TCP: tcpSocket Port: 8095
What happens, in order

# StorageClass ebs-sc tells 
Kubernetes to use the AWS EBS CSI driver and wait until the pod is scheduled.

# The PVC ebs-mysql-pv-claim requests 4Gi. 
When the MySQL pod is scheduled, an EBS volume is created and attached to that node, then mounted at /var/lib/mysql.

# The ConfigMap provides mysql_usermgmt.sql,
 mounted at /docker-entrypoint-initdb.d. On first container start (empty data dir), MySQL auto-runs SQL files there → creates DB usermgmt.

# The mysql Service is headless (clusterIP: None). 
DNS name mysql resolves directly to the MySQL pod IP (fine for a single replica).

# Usermgmt microservice gets DB connection details 
via env vars and reaches mysql:3306 through kube-DNS.

# The NodePort Service exposes the app on every node
at port 31231, mapping to container 8095. You can test with:
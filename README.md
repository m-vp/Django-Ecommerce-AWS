```bash
# 📌 Complete Step-by-Step Guide to Deploy a Django Application on AWS EKS

# ✅ Step 1: IAM Setup
# A. Create IAM Admin User
1. Go to IAM > Users > Add User
2. Username: admin-user
3. Enable Console access + Programmatic access
4. Attach policy: AdministratorAccess
5. Save access key and secret key

# B. Create IAM Roles
## EKS Node Role
- Trusted entity: EC2
- Policies:
  - AmazonEKSWorkerNodePolicy
  - AmazonEC2ContainerRegistryReadOnly
  - CloudWatchAgentServerPolicy
- Name: eks-node-role

## EC2 Bastion Role
- Trusted entity: EC2
- Policy: AmazonSSMManagedInstanceCore
- Name: bastion-ssm-role

## ALB Controller Role
- Enable OIDC provider in EKS > Cluster > Configuration > Associate OIDC
- IAM > Roles > Create Role
  - Trusted entity: Web identity
  - Provider: OIDC
  - Namespace: kube-system
  - Service account: aws-load-balancer-controller
  - Policy: AWSLoadBalancerControllerIAMPolicy

# 🌐 Step 2: VPC and Subnet Setup
1. Go to VPC > Create VPC > VPC and more wizard
2. Prefix: django
3. IPv4 CIDR: 10.0.0.0/16
4. AZs: 2
5. Public Subnets:
   - 10.0.1.0/24 (AZ1)
   - 10.0.2.0/24 (AZ2)
6. Private Subnets:
   - 10.0.3.0/24 (AZ1)
   - 10.0.4.0/24 (AZ2)
7. Enable DNS hostname/resolution
8. NAT Gateway: auto-allocate EIP

# 🔒 Step 3: Create Security Groups
## alb-sg
- Inbound: 80, 443 from 0.0.0.0/0
- Outbound: All

## eks-node-sg
- Inbound: HTTP/HTTPS from alb-sg
- Outbound: All

## rds-sg
- Inbound: PostgreSQL (5432) from eks-node-sg
- Outbound: All

## bastion-sg
- Inbound: SSH (22) from your IP
- Outbound: All

# 🚀 Step 4: Bastion Host Setup
1. EC2 > Launch Instance
2. AMI: Amazon Linux 2
3. Type: t3.micro
4. Subnet: public-subnet-a
5. Enable public IP
6. SG: bastion-sg
7. IAM Role: bastion-ssm-role
8. Access via EC2 Connect or SSM

# 🗃 Step 5: RDS PostgreSQL
1. RDS > Create DB
2. Engine: PostgreSQL
3. Identifier: django-db
4. User/pass: custom
5. Type: db.t3.medium
6. Storage: 20 GiB + autoscaling
7. VPC: django-vpc
8. Subnet: private-a + b
9. Public Access: No
10. SG: rds-sg

# 📦 Step 6: Push Docker Image to ECR
1. ECR > Create repository: django-app
2. On your machine:
aws ecr get-login-password | docker login --username AWS --password-stdin <ecr-url>
docker build -t django-app .
docker tag django-app:latest <ecr-url>/django-app:latest
docker push <ecr-url>/django-app:latest

# ☸️ Step 7: EKS Cluster & Node Group
1. EKS > Create Cluster
   - Name: django-cluster
   - VPC/Subnets: django-vpc, private-a+b
   - SG: eks-node-sg
   - Logging: All enabled
2. Create Node Group
   - Name: django-node-group
   - Role: eks-node-role
   - Type: t3.medium
   - Min: 2, Max: 4
   - Subnets: private-a + b

# 🧾 Step 8: Kubernetes YAMLs
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django
  template:
    metadata:
      labels:
        app: django
    spec:
      containers:
        - name: django
          image: <ecr-url>/django-app:latest
          ports:
            - containerPort: 8000
          env:
            - name: DB_NAME
              value: "your_db"
            - name: DB_USER
              value: "your_user"
            - name: DB_PASSWORD
              value: "your_password"
            - name: DB_HOST
              value: "<rds-endpoint>"
            - name: DB_PORT
              value: "5432"

# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: django-service
spec:
  selector:
    app: django
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
      nodePort: 30080
  type: NodePort

# ingress.yaml (optional)
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: django-ingress
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/target-type: ip
    kubernetes.io/ingress.class: alb
spec:
  rules:
    - http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: django-service
                port:
                  number: 80

# 📥 Step 9: Deploy to EKS
aws eks update-kubeconfig --name django-cluster
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml
kubectl get all
kubectl get ingress

# 🌍 Step 10: Manual ALB Setup (No Ingress)
1. Create Target Group:
   - Type: Instance
   - Port: 30080
   - VPC: django-vpc
   - Register EKS node private IPs
   - Health check: /
2. Create ALB:
   - Internet-facing
   - VPC: django-vpc
   - Subnets: public-a/b
   - SG: alb-sg
   - Listener: HTTP (80) → forward to target group

# 📊 Step 11: Monitoring and Notifications
## Enable CloudWatch Container Insights
- EKS > Cluster > Monitoring > Enable Container Insights

## View Logs
- CloudWatch > Logs > /aws/containerinsights/...

## SNS Alerts
1. SNS > Create Topic: django-alerts
2. Create Subscription: email

## CloudWatch Alarms
1. CloudWatch > Alarms > Create Alarm
2. Metric: CPUUtilization > 80% > 5min
3. Notify: SNS django-alerts

## CloudTrail
1. CloudTrail > Trails > Create trail
2. Store in S3, enable management events

## EventBridge Rule
1. EventBridge > Create Rule
2. Source: CloudWatch Alarm in ALARM state
3. Target: Lambda or SNS to restart pod

# 🛠 Step 12: Django DB Configuration
# settings.py
DATABASES = {
  'default': {
    'ENGINE': 'django.db.backends.postgresql',
    'NAME': os.getenv("DB_NAME"),
    'USER': os.getenv("DB_USER"),
    'PASSWORD': os.getenv("DB_PASSWORD"),
    'HOST': os.getenv("DB_HOST"),
    'PORT': os.getenv("DB_PORT", 5432),
  }
}

# Run migrations
python manage.py makemigrations
python manage.py migrate

# ✅ DONE! Django is running on AWS EKS with DB, ALB, Monitoring & Security!

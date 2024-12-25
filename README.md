# Building, Dockerizing, and Deploying a Node.js App on AWS EKS with CI/CD and Ingress ALB Integration

The project includes:
- **EKS Cluster**: Using **AWS Fargate** for serverless pods.
- **Hello World App**: A simple Node.js application containerized with **Docker**.
- **Ingress Controller**: Exposing the application via an **ALB**.
- **GitHub Actions** CI/CD Pipeline: Automating application build, test, and deployment.

### 1. Launch and connect to the EC2 Instance:
- Select the latest Ubuntu AMI.
- Choose a t2.medium or larger instance type (for sufficient resources).
- Assign a security group allowing: 
    - SSH (Port `22`)
    - HTTP (Port `80`)
    - NodeJS app (Port `3000`)
- Connect to the instance.
    - `ssh -i "your-key.pem" ubuntu@<instance-public-ip>`

### 2. Update the System & Install Pre-requisites
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install curl tar unzip -y

# Install Docker
sudo apt install -y docker.io
sudo systemctl enable docker
sudo systemctl start docker
sudo usermod -aG docker $USER

# Install Kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client

# Install Aws CLI
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
rm awscliv2.zip

# Configure awscli (Obtain ```Access-Key-ID``` and ```Secret-Access-Key``` from the AWS Management Console).
aws configure

# Install eksctl
curl -LO "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz"
tar -xzf eksctl_$(uname -s)_amd64.tar.gz
sudo mv eksctl /usr/local/bin
eksctl version
rm eksctl_$(uname -s)_amd64.tar.gz

# Install Helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### 3. Build and Push the Image to Docker Hub
```bash
docker build -t hello-world-app .
docker run -p 3000:3000 hello-world-app

# Open http://<ec2-public-ip-address>:3000 to confirm it works.

# Push the Image to Docker Hub:
docker tag hello-world-app <your-dockerhub-username>/hello-world-app:v1
docker push <your-dockerhub-username>/hello-world-app:v1
```

### 4. Cluster Setup with eksctl using Aws Fargate
```bash
eksctl create cluster --name demo-cluster --region us-east-1 --fargate

# This will:
# Create a VPC with public and private subnets.
# Set up Fargate profiles for namespaces (default and kube-system).
# Deploy a Kubernetes control plane managed by AWS.
# Configures networking for the cluster.
# --fargate: Sets up Fargate profiles to manage serverless pods (worker nodes).
# This step takes 10-15 minutes. 

# Verfiy Cluster
eksctl get cluster --name demo-cluster --region us-east-1

# Configuring kubectl for EKS:
aws eks update-kubeconfig --region us-east-1 --name demo-cluster

# Verify cluster access:
kubectl get nodes

# Create a namespace for the application:
kubectl create namespace hello-space
```

### 5. Create Custom Fargate profile dedicated to hello-space namespace
```bash
# # If you want to deploy resources in a custom namespace (e.g., hello-space), create a Fargate profile
# create a custom Fargate profile for the demo-cluster. 
# The profile allows Kubernetes pods in the hello-space namespace to run on AWS Fargate.
# this Fargate profile applies only to the game-2048 namespace.
# Pods in other namespaces (besides default and kube-system) won't run on Fargate unless explicitly configured.
eksctl create fargateprofile \
    --cluster demo-cluster \
    --region us-east-1 \
    --name fargate-profile-1 \
    --namespace hello-space

# Verify the Fargate profile creation:
eksctl get fargateprofile --cluster demo-cluster --region us-east-1

# This setup ensures that:
# System-level pods (e.g., CoreDNS) run in the kube-system namespace on Fargate.
# Application pods (e.g., the 2048 game app) run in the game-2048 namespace on Fargate.
```

### 6. Deploy the Application to EKS
```bash
# deploy.yaml specifies the application pod replicas and container image.
kubectl apply -f mainfests/hello-app/deploy.yaml -n hello-space
# service.yaml exposes pods internally using ClusterIP and externally using LoadBalancer or NodePort.
kubectl apply -f mainfests/hello-app/service.yaml -n hello-space

# Verify that pods and services are running:
kubectl get pods -n hello-space
kubectl get svc -n hello-space
kubectl get all -n hello-space
```

### 7. Setting Up Application Load Balancer (ALB) Ingress Controller which manages application load balancers for external traffic.
```bash
# Associate an IAM OIDC identity provider to integrate and authenticate Kubernetes service accounts with AWS IAM roles.

export cluster_name=demo-cluster
oidc_id=$(aws eks describe-cluster --name $cluster_name --query "cluster.identity.oidc.issuer" --output text | cut -d '/' -f 5)

# Check if there is an IAM OIDC provider configured already
aws iam list-open-id-connect-providers | grep $oidc_id | cut -d "/" -f4\n
# If not, run the below command to enable IAM OIDC Provider for the EKS cluster:
eksctl utils associate-iam-oidc-provider \
  --region us-east-1 \
  --cluster demo-cluster \
  --approve

# Download IAM policy
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json

# Create IAM Policy for the ALB controller:
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

# Create Service Account:
# Attach the policy to the Kubernetes service account:
eksctl create iamserviceaccount \
  --cluster=demo-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn=arn:aws:iam::<your-aws-account-id>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve
  --override-existing-serviceaccounts

# The ALB controller watches Kubernetes Ingress resources and automatically creates and configures an ALB for traffic routing.
# Public traffic routed via `ALB` → `Ingress Controller` → `Service` → `Pod`. 

# Deploy the ALB Controller using Helm:
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

# Install the controller:
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \            
  -n kube-system \
  --set clusterName=demo-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set us-east-1 \
  --set vpcId=<your-vpc-id>

# Verify Installation:
kubectl get pods -n kube-system
```

### 8. Apply the Ingress resource to expose the application via ALB
```bash
kubectl apply -f mainfests/hello-app/ingress.yaml -n hello-space

# Ingress Resource (ingress.yaml) exposes the app externally using hostname-based rules.

# Verify Ingress:
kubectl get ingress -n hello-space
```
### 9. Access the Application
- Access Application using the <EXTERNAL-IP> from the Ingress output.
    - `kubectl get ingress hello-world-ingress`
    - `http://<load-balancer-address>`

### 10. Monotir and Verify
```bash
# Check Resources
kubectl get pods --all-namespaces
kubectl get all -n hello-space

# Logs of a Pod
kubectl logs <pod-name> -n hello-space

# Check the Ingress Resource
kubectl get deploy -n kube-system
kubectl get deployment -n kube-system aws-load-balancer-controller
kubectl logs deployment/aws-load-balancer-controller -n kube-system

# View ALB in AWS Console
# Go to EC2 Console > Load Balancers
# Verify ALB setup and listeners.
```

### 11. Automate with CI/CD
- Store Secrets in GitHub: `Go to Settings > Secrets and variables > Actions and add`:
  - DOCKER_USERNAME
  - DOCKER_PASSWORD
  - AWS_ACCESS_KEY
  - AWS_SECRET_KEY
- Every push to the GitHub repository triggers the CI/CD pipeline, automating the build and deployment.

### 12. Cleanup Resources
```bash
eksctl delete cluster --name demo-cluster --region us-east-1

# Verify deletion:
eksctl get cluster --name demo-cluster --region us-east-1
```

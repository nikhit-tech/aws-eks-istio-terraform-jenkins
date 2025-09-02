# aws-eks-istio-terraform-jenkins

Here’s a hands-on lab you can follow end-to-end to stand up the stack on AWS EKS with Terraform, Jenkins, Docker, Kubernetes, Istio, ALB, Route 53, ExternalDNS, and Ansible, including a canary rollout. It’s designed so you can copy-paste, swap a few variables, and run.

What you’ll build

Infra with Terraform: VPC, subnets, EKS, node groups, ECR, IAM roles.

Cluster add-ons: AWS Load Balancer Controller, ExternalDNS, Istio.

CI/CD: Jenkins pipeline that builds a Docker image, pushes to ECR, applies Terraform, and deploys K8s + Istio.

Traffic: Route 53 → ALB → Istio IngressGateway → microservices on EKS.

Release: Canary via Istio VirtualService/DestinationRule. 

0) Prerequisites (one-time on your admin laptop)

AWS account with Admin (or equivalent) and a registered Route 53 hosted zone (e.g., dev.example.com).

Installed: awscli v2, kubectl, helm, terraform ≥1.5, docker, git, openssl, jq.

A Git repo you control (GitHub/GitLab/CodeCommit).

A Jenkins server (can be EC2 or Docker on your laptop). We’ll provision Jenkins with Ansible in Step 3.

Replace these variables everywhere below:

export AWS_REGION=us-west-2
export AWS_ACCOUNT_ID=111122223333
export CLUSTER_NAME=eks-devops-lab
export DOMAIN=dev.example.com           # Route 53 hosted zone
export APP_HOST=app.dev.example.com     # app DNS you want
export ECR_REPO=fullstack-app

1) Bootstrap: Terraform for EKS, VPC, ECR

Create a new folder infra/ and add these minimal files.

1.1 providers.tf
terraform {
  required_version = ">= 1.5.0"
  required_providers {
    aws = { source = "hashicorp/aws", version = "~> 5.0" }
    kubernetes = { source = "hashicorp/kubernetes", version = "~> 2.29" }
    helm = { source = "hashicorp/helm", version = "~> 2.12" }
  }
}

provider "aws" {
  region = var.region
}

# Kube + Helm providers are configured after EKS is created using data from aws_eks_cluster/auth

1.2 variables.tf
variable "region"        { default = "us-west-2" }
variable "cluster_name"  { default = "eks-devops-lab" }
variable "domain"        { default = "dev.example.com" }
variable "ecr_repo"      { default = "fullstack-app" }

1.3 main.tf (VPC, EKS, node group, ECR)

For brevity, this uses community modules.

locals {
  tags = { Project = "devops-lab" }
}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"
  name    = "${var.cluster_name}-vpc"
  cidr    = "10.20.0.0/16"
  azs     = ["${var.region}a","${var.region}b","${var.region}c"]
  private_subnets = ["10.20.1.0/24","10.20.2.0/24","10.20.3.0/24"]
  public_subnets  = ["10.20.101.0/24","10.20.102.0/24","10.20.103.0/24"]
  enable_nat_gateway = true
  tags = local.tags
}

module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"
  cluster_name    = var.cluster_name
  cluster_version = "1.29"
  subnet_ids      = module.vpc.private_subnets
  vpc_id          = module.vpc.vpc_id

  eks_managed_node_groups = {
    ng = {
      min_size     = 2
      max_size     = 6
      desired_size = 3
      instance_types = ["m6i.large"]
      ami_type       = "AL2_x86_64"
      capacity_type  = "ON_DEMAND"
    }
  }
  tags = local.tags
}

resource "aws_ecr_repository" "app" {
  name = var.ecr_repo
  image_scanning_configuration { scan_on_push = true }
  tags = local.tags
}

# ALB Controller IAM role for service account
module "alb_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.39"
  role_name             = "${var.cluster_name}-alb-controller"
  attach_load_balancer_controller_policy = true
  oidc_providers = {
    main = {
      provider_arn = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:aws-load-balancer-controller"]
    }
  }
}

# ExternalDNS IRSA
module "externaldns_irsa" {
  source  = "terraform-aws-modules/iam/aws//modules/iam-role-for-service-accounts-eks"
  version = "~> 5.39"
  role_name = "${var.cluster_name}-externaldns"
  attach_external_dns_policy = true
  oidc_providers = {
    main = {
      provider_arn = module.eks.oidc_provider_arn
      namespace_service_accounts = ["kube-system:external-dns"]
    }
  }
}

1.4 outputs.tf
output "cluster_name" { value = module.eks.cluster_name }
output "ecr_repo_url" { value = aws_ecr_repository.app.repository_url }

1.5 Apply
cd infra
terraform init
terraform apply -auto-approve -var="region=$AWS_REGION" -var="cluster_name=$CLUSTER_NAME" -var="ecr_repo=$ECR_REPO"
aws eks update-kubeconfig --name $CLUSTER_NAME --region $AWS_REGION
kubectl get nodes

2) Cluster add-ons: ALB Controller, ExternalDNS, Istio (via Helm)
2.1 AWS Load Balancer Controller
helm repo add eks https://aws.github.io/eks-charts
kubectl create ns kube-system || true
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=$CLUSTER_NAME \
  --set serviceAccount.create=true \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=$AWS_REGION \
  --set vpcId=$(terraform -chdir=infra output -raw vpc_id || echo) \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=$(terraform -chdir=infra output -raw alb_irsa_iam_role_arn)
kubectl -n kube-system rollout status deploy/aws-load-balancer-controller

2.2 ExternalDNS (assumes public hosted zone for $DOMAIN)
HOSTED_ZONE_ID=$(aws route53 list-hosted-zones-by-name --dns-name $DOMAIN --query 'HostedZones[0].Id' --output text | sed 's/\/hostedzone\///')

helm repo add bitnami https://charts.bitnami.com/bitnami
helm upgrade --install external-dns bitnami/external-dns \
  -n kube-system --create-namespace \
  --set provider=aws \
  --set aws.zoneType=public \
  --set txtOwnerId=$CLUSTER_NAME \
  --set policy=sync \
  --set domainFilters[0]=$DOMAIN \
  --set serviceAccount.create=true \
  --set serviceAccount.name=external-dns \
  --set serviceAccount.annotations."eks\.amazonaws\.com/role-arn"=$(terraform -chdir=infra output -raw externaldns_irsa_iam_role_arn)

2.3 Istio (demo profile is fine to start)
helm repo add istio https://istio-release.storage.googleapis.com/charts
kubectl create ns istio-system || true
helm upgrade --install istio-base istio/base -n istio-system
helm upgrade --install istiod istio/istiod -n istio-system --set meshConfig.enableTracing=true
helm upgrade --install istio-ingress istio/gateway -n istio-system
kubectl -n istio-system get pods

3) Jenkins + Ansible (bootstrap CI server)

Provision Jenkins quickly with Ansible (run from your admin box). Create ops/jenkins.yml:

- hosts: jenkins
  become: yes
  tasks:
    - name: Install deps
      apt:
        name: [openjdk-17-jre, docker.io, git, unzip, curl]
        state: present
        update_cache: yes
    - name: Add Jenkins repo & install
      shell: |
        curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | tee /usr/share/keyrings/jenkins-keyring.asc > /dev/null
        echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/ | tee /etc/apt/sources.list.d/jenkins.list > /dev/null
        apt update && apt install -y jenkins
    - name: Enable docker for jenkins
      user: name=jenkins groups=docker append=yes
    - name: Start Jenkins
      service: name=jenkins state=started enabled=yes


Your inventory.ini:

[jenkins]
<YOUR_JENKINS_EC2_PUBLIC_IP> ansible_user=ubuntu


Run:

ansible-playbook -i ops/inventory.ini ops/jenkins.yml


In Jenkins UI:

Install plugins: Pipeline, Git, Blue Ocean, Credentials Binding, CloudBees AWS Credentials.

Add credentials:

AWS access key with ECR/EKS permissions.

DockerHub (if using DockerHub), otherwise use ECR only.

Kubernetes kubeconfig (optional: we’ll shell out via AWS CLI).

4) App repo: Dockerfile, K8s, Istio, Jenkinsfile
4.1 Dockerfile
FROM public.ecr.aws/docker/library/nginx:alpine
COPY index.html /usr/share/nginx/html/index.html

4.2 index.html
<!doctype html><html><body>
<h1>Full-Stack App v1</h1>
</body></html>

4.3 Kubernetes manifests (k8s/)

namespace.yaml

apiVersion: v1
kind: Namespace
metadata:
  name: app
  labels:
    istio-injection: enabled


deployment.yaml (labels to support canary)

apiVersion: apps/v1
kind: Deployment
metadata:
  name: web
  namespace: app
spec:
  replicas: 3
  selector:
    matchLabels: { app: web, version: v1 }
  template:
    metadata:
      labels: { app: web, version: v1 }
    spec:
      containers:
      - name: web
        image: ${ECR_URI}:v1
        ports:
        - containerPort: 80


service.yaml

apiVersion: v1
kind: Service
metadata:
  name: web
  namespace: app
spec:
  selector: { app: web }
  ports:
    - port: 80
      targetPort: 80

4.4 Istio manifests (istio/)

gateway.yaml (expose through Istio)

apiVersion: networking.istio.io/v1beta1
kind: Gateway
metadata:
  name: app-gw
  namespace: app
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "${APP_HOST}"


destinationrule.yaml

apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: web
  namespace: app
spec:
  host: web.app.svc.cluster.local
  subsets:
  - name: v1
    labels: { version: v1 }
  - name: v2
    labels: { version: v2 }


virtualservice.yaml (start 100% to v1)

apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: web
  namespace: app
spec:
  gateways: ["app-gw"]
  hosts: ["${APP_HOST}"]
  http:
  - route:
    - destination: { host: web.app.svc.cluster.local, subset: v1 }
      weight: 100

4.5 Ingress to provision ALB for the Istio ingress gateway

This Kubernetes Ingress targets the istio-ingressgateway Service and lets ALB front it; ExternalDNS creates your Route 53 record.

apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: istio-alb
  namespace: istio-system
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    external-dns.alpha.kubernetes.io/hostname: ${APP_HOST}
spec:
  rules:
  - host: ${APP_HOST}
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: istio-ingressgateway
            port:
              number: 80

4.6 Jenkinsfile (build → push → deploy)
pipeline {
  agent any
  environment {
    AWS_REGION = "${AWS_REGION}"
    ECR_REPO  = "${ECR_REPO}"
    CLUSTER   = "${CLUSTER_NAME}"
    APP_HOST  = "${APP_HOST}"
    ECR_URI   = "${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}"
  }
  stages {
    stage('Checkout'){ steps { checkout scm } }

    stage('Login ECR'){
      steps {
        sh '''
        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_URI
        '''
      }
    }

    stage('Build & Push'){
      steps {
        sh '''
        docker build -t $ECR_URI:v1 .
        docker push $ECR_URI:v1
        '''
      }
    }

    stage('Kubeconfig'){
      steps {
        sh 'aws eks update-kubeconfig --name $CLUSTER --region $AWS_REGION'
      }
    }

    stage('Deploy K8s + Istio'){
      steps {
        sh '''
        ECR_URI=$ECR_URI envsubst < k8s/deployment.yaml | kubectl apply -f -
        kubectl apply -f k8s/namespace.yaml
        kubectl apply -f k8s/service.yaml

        APP_HOST=$APP_HOST envsubst < istio/gateway.yaml | kubectl apply -f -
        kubectl apply -f istio/destinationrule.yaml
        APP_HOST=$APP_HOST envsubst < istio/virtualservice.yaml | kubectl apply -f -

        APP_HOST=$APP_HOST envsubst < alb/ingress.yaml | kubectl apply -f -
        '''
      }
    }
  }
}


Commit and push your repo. Create a Jenkins Multibranch Pipeline or a simple Pipeline using this Jenkinsfile.

5) Verify DNS → ALB → Istio → App
kubectl -n istio-system get ingress istio-alb
# Get ADDRESS column (ALB DNS). ExternalDNS should create a Route53 A/ALIAS for $APP_HOST.
dig +short $APP_HOST
# Should resolve to ALB. Then:
curl -I http://$APP_HOST


Open http://app.dev.example.com in a browser—expect “Full-Stack App v1”.

6) Canary rollout with Istio (10% v2 → 50% → 100%)
6.1 Build and deploy v2

Update index.html:

<h1>Full-Stack App v2</h1>


Build & push a v2 image and a v2 deployment:

docker build -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:v2 .
docker push  $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/$ECR_REPO:v2

# v2 deployment (k8s/deployment-v2.yaml)
cat <<'EOF' | envsubst | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-v2
  namespace: app
spec:
  replicas: 2
  selector:
    matchLabels: { app: web, version: v2 }
  template:
    metadata:
      labels: { app: web, version: v2 }
    spec:
      containers:
      - name: web
        image: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO}:v2
        ports: [ { containerPort: 80 } ]
EOF

6.2 Shift 10% traffic to v2
# istio/virtualservice-canary-10.yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: web
  namespace: app
spec:
  gateways: ["app-gw"]
  hosts: ["${APP_HOST}"]
  http:
  - route:
    - destination: { host: web.app.svc.cluster.local, subset: v1 }
      weight: 90
    - destination: { host: web.app.svc.cluster.local, subset: v2 }
      weight: 10


Apply (then gradually move to 50/50, then 0/100 when happy):

APP_HOST=$APP_HOST envsubst < istio/virtualservice-canary-10.yaml | kubectl apply -f -
# Later: adjust weights to 50/50, then 0/100 to complete rollout.

7) Observability & health checks (quick start)

Check ALB health:
kubectl -n kube-system logs deploy/aws-load-balancer-controller | tail -n 50

See ExternalDNS syncing:
kubectl -n kube-system logs deploy/external-dns | tail -n 100

Istio metrics with kubectl -n istio-system get pods, consider installing Kiali/Grafana later.

Simple traffic sampling: for i in {1..20}; do curl -s http://$APP_HOST | grep Full-Stack; done

8) Cleanup (to avoid surprise costs)
kubectl delete ingress istio-alb -n istio-system
kubectl delete ns app
helm -n istio-system uninstall istio-ingress istiod istio-base || true
helm -n kube-system uninstall external-dns aws-load-balancer-controller || true
terraform -chdir=infra destroy -auto-approve

Notes & Options

You can move Jenkins into the cluster with Jenkins Helm chart; the pipeline remains the same.

Swap Jenkins for GitLab CI easily—stages are identical (build → push → deploy).

For HTTPS: add ACM cert ARN via ALB annotations and use Istio Gateway with TLS server.

For blue/green instead of canary, use two VirtualService routes with 0/100 switches.

If you want, I can package this lab as a Git repo (with all files in place) so you can clone and run with minimal edits.

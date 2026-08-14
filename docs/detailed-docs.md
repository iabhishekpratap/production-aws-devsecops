# Detailed Setup Guide — Production-Grade AWS DevSecOps Platform

This guide walks through provisioning, configuring, and deploying the full stack in this repo end to end: Jenkins server → Terraform-driven VPC/EKS → CI/CD (SonarQube + OWASP + Trivy + ECR) → GitOps deploy via Argo CD → Ingress/custom domain → Prometheus/Grafana monitoring.

---

## Architecture

- **App:** 3-tier MERN app — React frontend, Express/Node backend, MongoDB (public Docker image).
- **Infra:** Terraform provisions a VPC with an EKS cluster (2 worker nodes) in private subnets, plus a public jump server to reach it.
- **CI:** Jenkins pipeline — SonarQube scan → OWASP Dependency-Check → Trivy image scan → build & push to private ECR.
- **CD:** Argo CD watches the Kubernetes manifests in Git and syncs them to the cluster (GitOps).
- **Exposure:** AWS Load Balancer Controller + Ingress resource, optionally bound to a custom domain via Route 53.
- **Observability:** kube-prometheus-stack (Prometheus + Grafana) via Helm.

See `assets/Three-Tier.gif` in the repo for the visual architecture diagram.

## Prerequisites

- An AWS account with an IAM user/role that can create IAM roles, VPCs, EKS clusters, EC2, ECR, S3, DynamoDB, and Route 53 records.
- An AWS Access Key ID / Secret Access Key for that user (or use an instance profile — see Step 1).
- A GitHub account and a **fine-scoped Personal Access Token** (repo + workflow scopes) — generate it fresh, don't reuse an old one.
- A registered domain if you want the custom-domain step (optional).
- Basic familiarity with `kubectl`, `helm`, `terraform`, and AWS CLI.

---



## Step 1 — Launch the Jenkins Master EC2

Provision this manually (or via `Jenkins-Server-TF`) with:

| Setting              | Value                                                                                                                                                  |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Instance type        | `t2.2xlarge` (Jenkins + Terraform + Docker + Trivy + SonarQube client all run here)                                                                    |
| Storage              | 30 GB gp3                                                                                                                                              |
| Key pair             | None — access via **AWS Session Manager** instead of SSH                                                                                               |
| IAM instance profile | Attach a role with admin access (or `AmazonSSMManagedInstanceCore` + the permissions Terraform needs) so Session Manager works without opening port 22 |
| Security group       | Inbound: `8080` (Jenkins UI), `9090` (SonarQube)                                                                                                       |

Connect via **EC2 → Connect → Session Manager** (no SSH key needed).

Once connected, install Jenkins, Docker, Terraform, and AWS CLI on the box (a user-data script or the `Jenkins-Server-TF` module can automate this — see that folder for the exact provisioning script).

Get the initial Jenkins admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Log in at `http://<jenkins-ec2-public-ip>:8080`.

## Step 2 — Install Jenkins Plugins & Configure Tools

**Manage Jenkins → Plugins → Available**, install:

- Stage View
- AWS Credentials
- Pipeline: AWS Steps
- Terraform
- Docker, Docker Commons, Docker Pipeline, Docker API, docker-build-step
- Eclipse Temurin installer
- NodeJS
- OWASP Dependency-Check
- SonarQube Scanner

**Manage Jenkins → Tools**, configure:

- **Terraform** — point to the install directory (run `whereis terraform` on the Jenkins box to find it).
- **NodeJS** — name it `nodejs`, pick a version, let Jenkins auto-install.
- **SonarQube Scanner** — name it `sonar-scanner`, let Jenkins auto-install.
- **Eclipse Temurin (JDK)** — needed by the Sonar scanner.

**Manage Jenkins → System** → add a SonarQube server entry (name + URL + auth token — token comes from Step 8) so the pipeline's `withSonarQubeEnv` step resolves.

## Step 3 — Add Jenkins Credentials

**Manage Jenkins → Credentials → System → Global credentials**, add:

| Kind              | ID                                                     | Contents                                                                |
| ----------------- | ------------------------------------------------------ | ----------------------------------------------------------------------- |
| Username/password | `github-creds` (name it what your Jenkinsfile expects) | GitHub username + your **new** fine-scoped PAT                          |
| Secret text       | `sonar-token`                                          | Token generated in Step 8                                               |
| Secret text       | `ACCOUNT_ID`                                           | Your AWS account ID                                                     |
| AWS credentials   | (used by Terraform/AWS steps)                          | AWS Access Key + Secret Key for a scoped IAM user — never your root key |
| Secret text       | `ECR_REPO01`                                           | Frontend ECR repo name                                                  |
| Secret text       | `ECR_REPO02`                                           | Backend ECR repo name                                                   |

> Note: the original build notes for this project embedded a real GitHub PAT, Jenkins password, and Sonar token in plaintext. If any of those were ever pushed to a public repo or shared, **rotate them immediately** — treat this as a live incident, not just cleanup.

## Step 4 — Provision the VPC + EKS Cluster via Jenkins/Terraform

Before running `terraform apply`, create the remote state backend once, manually:

```bash
aws s3 mb s3://absk-tf-bucket --region ap-south-1
aws dynamodb create-table \
  --table-name Lock-Files \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-south-1
```

(Use your own bucket/table names and reference them in the Terraform backend config.)

In Jenkins, create a pipeline job pointing at the infra Terraform code (`Jenkins-Pipeline-Code`), parameterized to run `plan` then, on a **Build with Parameters** run, `apply`. This provisions:

- A VPC (e.g. `dev-medium-vpc`) with public + private subnets
- An EKS cluster (e.g. `dev-medium-eks-cluster`) with 2 worker nodes in the private subnets

## Step 5 — Launch a Jump Server & Connect to EKS

The EKS cluster and worker nodes sit in private subnets, so you need a bastion/jump host in a **public** subnet of the same VPC:

- Subnet: any public subnet from `dev-medium-vpc`
- Security group: new SG allowing SSH/SSM from your IP
- User data: install `aws-cli`, `kubectl`, `helm`, `eksctl`

From the jump server, configure `kubectl`:

```bash
aws configure   # or attach an instance profile instead of static keys
aws eks update-kubeconfig --name dev-medium-eks-cluster --region ap-south-1

kubectl get nodes
kubectl get pods -A
```

## Step 6 — Install the AWS Load Balancer Controller (IRSA)

A pod can't create AWS resources (like an ALB) on its own — it needs an IAM role tied to its Kubernetes service account (**IRSA**). On the jump server:

```bash
# 1. Fetch and create the IAM policy the controller needs
curl -O https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.5.4/docs/install/iam_policy.json
aws iam create-policy \
  --policy-name AWSLoadBalancerControllerIAMPolicy \
  --policy-document file://iam_policy.json

# 2. Associate an OIDC provider with the cluster (required for IRSA)
eksctl utils associate-iam-oidc-provider \
  --region=ap-south-1 \
  --cluster=dev-medium-eks-cluster \
  --approve

# 3. Create the service account + IAM role binding
eksctl create iamserviceaccount \
  --cluster=dev-medium-eks-cluster \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --role-name AmazonEKSLoadBalancerControllerRole \
  --attach-policy-arn arn:aws:iam::<ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
  --approve \
  --region=ap-south-1 \
  --override-existing-serviceaccounts

kubectl get sa -n kube-system   # confirm aws-load-balancer-controller exists
```

Install the controller via Helm:

```bash
sudo snap install helm --classic
helm repo add eks https://aws.github.io/eks-charts
helm repo update eks

helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=dev-medium-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller

kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

If the pod crash-loops (commonly a missing region/VPC ID), re-run with explicit values:

```bash
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=dev-medium-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=ap-south-1 \
  --set vpcId=<your-vpc-id> \
  -n kube-system

kubectl get deployment -n kube-system aws-load-balancer-controller   # expect 2/2 ready
```

## Step 7 — Install Argo CD

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.4.7/manifests/install.yaml
kubectl get pods -n argocd
```

Expose the Argo CD UI via a LoadBalancer:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd   # grab the EXTERNAL-IP / DNS
```

Get the initial admin password:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 --decode
```

Log in at `https://<argocd-lb-dns>` with user `admin` and the decoded password — then **change it immediately** from the UI/CLI.

In **Settings → Repositories**, connect this GitHub repo via SSH (deploy key) so Argo CD can read the manifests in `Kubernetes-Manifests-file/`.

## Step 8 — Set Up SonarQube

Run it on the Jenkins EC2 (open port `9000` in the security group):

```bash
docker run -d --name sonarqube -p 9000:9000 sonarqube:community
```

1. Log in at `http://<jenkins-ip>:9000` (default `admin`/`admin`, change it).
2. **My Account → Security** → generate a token, save it as the `sonar-token` Jenkins credential (Step 3).
3. **Administration → Webhooks** → add a webhook named `jenkins` pointing at `http://<jenkins-ip>:8080/sonarqube-webhook/` (no secret needed).
4. Create two projects: `frontend` and `backend` (project key = project name).

Ad-hoc local scan (optional, to sanity-check before wiring into Jenkins):

```bash
sonar-scanner \
  -Dsonar.projectKey=frontend \
  -Dsonar.sources=. \
  -Dsonar.host.url=http://<sonarqube-ip>:9000 \
  -Dsonar.token=<your-sonar-token>
```

Repeat with `projectKey=backend` from the backend source directory.

## Step 9 — Create ECR Repositories

If not already created by Terraform, create two private ECR repos:

```bash
aws ecr create-repository --repository-name frontend --region ap-south-1
aws ecr create-repository --repository-name backend --region ap-south-1
```

Reference these repo names in the `ECR_REPO01` / `ECR_REPO02` Jenkins credentials from Step 3 — the pipeline needs them to know where to push images.

## Step 10 — Run the CI/CD Pipeline in Jenkins

Create a new Pipeline job pointing at the app Jenkinsfile in `Jenkins-Pipeline-Code/`. Each run should:

1. Checkout source (`Application-Code`)
2. SonarQube static analysis (frontend + backend)
3. OWASP Dependency-Check (fails/warns on known-vulnerable dependencies)
4. File-system scan
5. Build Docker images (using the existing Dockerfiles in `Application-Code`)
6. Push images to ECR
7. Trivy container-image scan on the pushed images
8. Bump the image tag/version in the Kubernetes manifests (so Argo CD picks up the new version)

Trigger a build and watch the Stage View to confirm each gate passes.

## Step 11 — Deploy the App via Argo CD

On the jump server, create the app namespace and deploy the database first (it has no dependency on the app images):

```bash
kubectl create ns three-tier
```

The EKS CSI driver for EBS provisions a PersistentVolume for MongoDB in the same AZ as the pod and attaches it to that node automatically — no manual EBS setup needed as long as the CSI add-on is enabled on the cluster.

In the Argo CD UI, create three Applications pointing at the connected repo, each targeting a path under `Kubernetes-Manifests-file/`:

1. **Database** — deploys MongoDB into the `three-tier` namespace
2. **Backend** — update `Kubernetes-Manifests-file/Backend/deployment.yaml` with your ECR image URI before/after syncing
3. **Frontend** — same, using `Kubernetes-Manifests-file/Frontend/deployment.yaml`

Sync each Application and confirm pods come up healthy:

```bash
kubectl get pods -n three-tier
```

## Step 12 — Expose the App (Ingress + Custom Domain)

A `ClusterIP` service isn't reachable from outside the cluster. Create a fourth Argo CD Application for the **Ingress** resource (manifest lives alongside the frontend/backend manifests) — the AWS Load Balancer Controller from Step 6 will provision an ALB for it automatically.

```bash
kubectl get ingress -n three-tier   # grab the ALB's DNS name once ADDRESS populates
```

You'll get something like `k8s-threetie-mainlb-xxxxxxxxxx-xxxxxxxxxx.ap-south-1.elb.amazonaws.com`.

**Optional — custom domain via Route 53:**

| Field  | Value                  |
| ------ | ---------------------- |
| Type   | Alias (A record)       |
| Name   | `@` (or a subdomain)   |
| Target | The ALB DNS name above |
| TTL    | 300                    |

## Step 13 — Monitoring with Prometheus & Grafana

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

helm install prometheus prometheus-community/kube-prometheus-stack --namespace default
```

> If pods stay `Pending`, check your node's pod-limit and PVC provisioning first — this is the most common failure mode. If problems persist, try reinstalling Prometheus and Grafana as separate Helm releases rather than the combined stack.

Expose Grafana and Prometheus externally:

```bash
kubectl patch svc prometheus-grafana -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc prometheus-grafana   # EXTERNAL-IP is your Grafana URL, port 80

kubectl patch svc prometheus-kube-prometheus-prometheus -p '{"spec": {"type": "LoadBalancer"}}'
# Prometheus UI is on :9090 of that LB's DNS
```

Grafana default credentials come from a generated secret — decode them rather than assuming `admin/admin`:

```bash
kubectl get secret prometheus-grafana -o jsonpath="{.data.admin-user}" | base64 --decode; echo
kubectl get secret prometheus-grafana -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

**Change this password on first login.** In Grafana, add Prometheus as a data source (usually pre-wired by the chart) and import a Kubernetes dashboard by ID (e.g. 15757 or 6417) to get pod/node metrics immediately.

---

## Troubleshooting

- **AWS Load Balancer Controller crash-looping** → almost always a missing `region` or `vpcId` value; re-run the `helm upgrade` with both set explicitly (Step 6).
- **Prometheus/Grafana pods stuck Pending** → check `kubectl describe pod` for pod-limit or PVC binding errors; the EBS CSI driver add-on must be enabled on the EKS cluster.
- **Argo CD app stuck `OutOfSync` after image push** → confirm the Jenkins pipeline actually committed the new image tag to the manifests repo; Argo CD only reacts to Git, not to ECR pushes directly.
- **`kubectl` can't reach the cluster from the jump server** → re-run `aws eks update-kubeconfig`, and confirm the jump server's IAM role/instance profile has `eks:DescribeCluster` and cluster access (via EKS access entries or the aws-auth ConfigMap).
- **SonarQube webhook never fires in Jenkins** → verify the webhook URL matches the Jenkins EC2's current IP/DNS exactly and that port 8080 is reachable from the SonarQube host.

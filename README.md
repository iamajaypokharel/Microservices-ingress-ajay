# Deployment of a Microservices Application using Ingress Controller

End-to-end DevOps project: a microservices application is built and pushed with **Jenkins + Docker**, deployed to **AWS EKS (Kubernetes)** behind an **NGINX Ingress Controller**, delivered with **Argo CD (GitOps)**, and monitored with **Prometheus + Grafana**.

> **Author:** Ajay Pokharel
> **Repository:** https://github.com/iamajaypokharel/Microservices-ingress-ajay
> **Reference:** Based on the DevOps project by Kastro Kiran V

---

## Table of Contents

1. [Architecture](#architecture)
2. [Tech Stack](#tech-stack)
3. [Prerequisites](#prerequisites)
4. [Step 1: Basic Setup](#step-1-basic-setup)
5. [Step 2: Install Jenkins and Docker](#step-2-install-jenkins-and-docker)
6. [Step 3: Configure Jenkins](#step-3-configure-jenkins)
7. [Step 4: Create the EKS Cluster and Ingress Controller](#step-4-create-the-eks-cluster-and-ingress-controller)
8. [Step 5: Jenkins Pipeline](#step-5-jenkins-pipeline)
9. [Step 6: Monitoring with Prometheus](#step-6-monitoring-with-prometheus)
10. [Step 7: Grafana Dashboards](#step-7-grafana-dashboards)
11. [Step 8: Argo CD Deployment](#step-8-argo-cd-deployment)
12. [Verify the Application](#verify-the-application)
13. [Cleanup](#cleanup)
14. [Troubleshooting](#troubleshooting)
15. [Security Notes](#security-notes)

---

## Architecture

```
Developer ──push──▶ GitHub ──webhook/poll──▶ Jenkins (CI)
                                               │  build, test, docker build
                                               ▼
                                          Docker Hub (images)
                                               │
                     GitHub (K8s manifests) ◀──┘ update image tag
                               │
                               ▼
                           Argo CD (CD / GitOps)
                               │ sync
                               ▼
        ┌──────────────── AWS EKS Cluster ────────────────┐
        │  NGINX Ingress Controller (AWS Load Balancer)   │
        │        │ path-based routing                     │
        │   ┌────┴─────┬───────────┬──────────┐           │
        │ Service A  Service B   Service C  Frontend ...  │
        └─────────────────────────────────────────────────┘

Monitoring VM: Prometheus ◀── Node Exporter, Jenkins metrics ──▶ Grafana
```

## Tech Stack

| Area | Tool |
|---|---|
| Source control | Git, GitHub |
| CI | Jenkins (Pipeline as Code via `Jenkinsfile`) |
| Containers | Docker, Docker Hub |
| Orchestration | Kubernetes on AWS EKS (`eksctl`) |
| Traffic routing | NGINX Ingress Controller |
| CD / GitOps | Argo CD (installed with Helm) |
| Monitoring | Prometheus, Node Exporter, Grafana |
| Cloud | AWS (EC2, EKS, IAM, ELB) |

## Prerequisites

- AWS account with permission to create IAM users, EC2 and EKS resources
- Docker Hub account
- GitHub account
- Basic knowledge of Linux, Docker and Kubernetes

---

## Step 1: Basic Setup

### 1.1 Push the code from local to remote

Use a GitHub **Personal Access Token (PAT)** for HTTPS authentication.

1. GitHub → **Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token**
2. Select scopes: `repo` (and `workflow` if you use GitHub Actions)
3. Copy the token (it is shown only once)

```bash
git remote set-url origin https://github.com/iamajaypokharel/Microservices-ingress-ajay.git
git push -u origin master
# When prompted: username = your GitHub username, password = your PAT
```

> Avoid embedding the token in the remote URL (`https://user:token@github.com/...`). It is stored in plain text in `.git/config` and shell history. Use a credential helper instead: `git config --global credential.helper store`.

### 1.2 Launch the Ingress Server VM

| Setting | Value |
|---|---|
| Name | `Ingress-Server` |
| OS | Ubuntu 24.04 |
| Instance type | `t2.large` |
| Storage | 28 GB |

Open these ports in the attached Security Group:

| Type | Protocol | Port | Purpose |
|---|---|---|---|
| SSH | TCP | 22 | Remote access |
| HTTP | TCP | 80 | Web traffic |
| HTTPS | TCP | 443 | Secure web traffic |
| SMTP | TCP | 25 | Email notifications |
| SMTPS | TCP | 465 | Secure email |
| Custom TCP | TCP | 3000-10000 | Jenkins (8080), Grafana (3000), apps |
| Custom TCP | TCP | 6443 | Kubernetes API server |
| Custom TCP | TCP | 30000-32767 | Kubernetes NodePort range |

> For anything beyond a demo, restrict source IPs instead of using `0.0.0.0/0`.

---

## Step 2: Install Jenkins and Docker

Connect to the `Ingress-Server` over SSH.

### 2.1 Install Jenkins

Create `jenkins.sh`:

```bash
#!/bin/bash
# Update system
sudo apt update -y

# Install dependencies
sudo apt install -y fontconfig openjdk-17-jre-headless wget gnupg2

# Add the Jenkins GPG key
wget -O- https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key | \
    gpg --dearmor | sudo tee /usr/share/keyrings/jenkins-keyring.gpg > /dev/null

# Add the Jenkins repository
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.gpg] https://pkg.jenkins.io/debian-stable binary/" | \
    sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null

# Install and start Jenkins
sudo apt update -y
sudo apt install jenkins -y
sudo systemctl start jenkins
sudo systemctl enable jenkins
sudo systemctl status jenkins
```

```bash
chmod +x jenkins.sh
./jenkins.sh
```

Jenkins runs on port **8080** (already open in the security group).

### 2.2 Install Docker

Create `docker.sh`:

```bash
#!/bin/bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl

# Docker GPG key
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Docker repository
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

```bash
chmod +x docker.sh
./docker.sh
docker --version
```

---

## Step 3: Configure Jenkins

### 3.1 Unlock Jenkins

Open `http://<ingress-server-public-ip>:8080` and get the initial admin password:

```bash
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

Install the suggested plugins and create the admin user.

### 3.2 Install plugins

**Manage Jenkins → Plugins → Available plugins**, install:

- Docker, Docker Commons, Docker Pipeline, Docker API, docker-build-step
- AWS Credentials
- Pipeline: Stage View
- Kubernetes, Kubernetes CLI, Kubernetes Client API, Kubernetes Credentials
- Config File Provider
- Prometheus metrics

### 3.3 Create credentials

**Manage Jenkins → Credentials → Global → Add Credentials**

| ID | Type | Contents |
|---|---|---|
| `dockerhub-creds` | Username with password | Docker Hub username and access token |
| `aws-creds` | AWS Credentials | AWS Access Key ID and Secret Access Key |

---

## Step 4: Create the EKS Cluster and Ingress Controller

### 4.1 Create an IAM user

Do not create the cluster with the root account. Create a dedicated IAM user (e.g. `eks-admin`).

### 4.2 Attach policies

Managed policies:

- `AmazonEC2FullAccess`
- `AmazonEKS_CNI_Policy`
- `AmazonEKSClusterPolicy`
- `AmazonEKSWorkerNodePolicy`
- `AWSCloudFormationFullAccess`
- `IAMFullAccess`

Inline policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "VisualEditor0",
      "Effect": "Allow",
      "Action": "eks:*",
      "Resource": "*"
    }
  ]
}
```

### 4.3 Create access keys

IAM → Users → your user → **Security credentials → Create access key**. Save the keys; they are used for `aws configure` and the `aws-creds` Jenkins credential.

### 4.4 Install AWS CLI

```bash
sudo apt update
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
sudo apt install unzip -y
unzip awscliv2.zip
sudo ./aws/install

aws configure
```

### 4.5 Install kubectl

```bash
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl
sudo mv kubectl /usr/local/bin/
kubectl version --client
```

> The original reference used a kubectl 1.19.6 binary from 2021. Use a recent version that is within one minor version of your EKS cluster.

### 4.6 Install eksctl

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

### 4.7 Create the EKS cluster

```bash
eksctl create cluster \
  --name kastro-cluster \
  --region us-east-1 \
  --node-type t2.medium \
  --zones us-east-1a,us-east-1b
```

This takes roughly 15 to 20 minutes. Confirm access:

```bash
kubectl get nodes
```

### 4.8 Give Jenkins access to Docker

```bash
sudo usermod -aG docker jenkins
sudo systemctl restart docker
sudo systemctl restart jenkins
```

### 4.9 Install the NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/aws/deploy.yaml

# Wait for the pods
kubectl get pods -n ingress-nginx

# Get the external address of the ingress load balancer
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

The `EXTERNAL-IP` (an AWS ELB hostname) is the public entry point for your application. Your Ingress rules route paths such as `/`, `/api/...` to the correct microservices.

---

## Step 5: Jenkins Pipeline

1. Jenkins → **New Item** → name it (e.g. `microservices-ingress-pipeline`) → **Pipeline**
2. Under **Pipeline**, choose **Pipeline script from SCM**
   - SCM: Git
   - Repository URL: `https://github.com/iamajaypokharel/Microservices-ingress-ajay.git`
   - Branch: `*/master`
   - Script Path: `Jenkinsfile`
3. Save and click **Build Now**

The pipeline logic lives in the [`Jenkinsfile`](./Jenkinsfile) in this repo. Typical stages:

1. Checkout source from GitHub
2. Build Docker images for each microservice
3. Push images to Docker Hub (using `dockerhub-creds`)
4. Configure `kubectl` for EKS (using `aws-creds`)
5. Deploy the Kubernetes manifests (Deployments, Services, Ingress)

---

## Step 6: Monitoring with Prometheus

Launch a second VM:

| Setting | Value |
|---|---|
| Name | `Monitoring Server` |
| OS | Ubuntu 22.04 |
| Instance type | `t2.medium` |
| Ports to open | 22, 9090 (Prometheus), 9100 (Node Exporter), 3000 (Grafana) |

### 6.1 Install Prometheus

Create a system user:

```bash
sudo apt update
sudo useradd --system --no-create-home --shell /bin/false prometheus
```

Download and install:

```bash
wget https://github.com/prometheus/prometheus/releases/download/v2.47.1/prometheus-2.47.1.linux-amd64.tar.gz
tar -xvf prometheus-2.47.1.linux-amd64.tar.gz
sudo mkdir -p /data /etc/prometheus
cd prometheus-2.47.1.linux-amd64/

sudo mv prometheus promtool /usr/local/bin/
sudo mv consoles/ console_libraries/ /etc/prometheus/
sudo mv prometheus.yml /etc/prometheus/prometheus.yml
sudo chown -R prometheus:prometheus /etc/prometheus/ /data/

cd ~
rm -rf prometheus-2.47.1.linux-amd64.tar.gz
prometheus --version
```

Create the systemd service `/etc/systemd/system/prometheus.service`:

```ini
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/prometheus \
  --config.file=/etc/prometheus/prometheus.yml \
  --storage.tsdb.path=/data \
  --web.console.templates=/etc/prometheus/consoles \
  --web.console.libraries=/etc/prometheus/console_libraries \
  --web.listen-address=0.0.0.0:9090 \
  --web.enable-lifecycle

[Install]
WantedBy=multi-user.target
```

Start it:

```bash
sudo systemctl enable prometheus
sudo systemctl start prometheus
sudo systemctl status prometheus
```

Open `http://<monitoring-ip>:9090`. Use **http**, not https. Under **Status → Targets** you should see `prometheus (1/1 up)`.

### 6.2 Install Node Exporter

```bash
sudo useradd --system --no-create-home --shell /bin/false node_exporter
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar -xvf node_exporter-1.6.1.linux-amd64.tar.gz
sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
rm -rf node_exporter*
node_exporter --version
```

Create `/etc/systemd/system/node_exporter.service`:

```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target
StartLimitIntervalSec=500
StartLimitBurst=5

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=on-failure
RestartSec=5s
ExecStart=/usr/local/bin/node_exporter --collector.logind

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
sudo systemctl status node_exporter
```

### 6.3 Integrate Jenkins and Node Exporter with Prometheus

Edit `/etc/prometheus/prometheus.yml`:

```yaml
scrape_configs:
  - job_name: "prometheus"
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "node_exporter"
    static_configs:
      - targets: ["<MONITORING_VM_IP>:9100"]

  - job_name: "jenkins"
    metrics_path: "/prometheus"
    static_configs:
      - targets: ["<JENKINS_IP>:8080"]
```

Replace `<MONITORING_VM_IP>` and `<JENKINS_IP>`. Keep port `9100` for Node Exporter even though Prometheus itself uses `9090`.

Validate and reload:

```bash
promtool check config /etc/prometheus/prometheus.yml
curl -X POST http://localhost:9090/-/reload
```

Open `http://<monitoring-ip>:9090/targets`. If Node Exporter shows `0/1`, open port **9100** in the Monitoring VM security group. All three targets (`prometheus`, `node_exporter`, `jenkins`) should now be **UP**.

---

## Step 7: Grafana Dashboards

### 7.1 Install Grafana on the Monitoring Server

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https software-properties-common wget

# Add Grafana GPG key and repository
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list

sudo apt-get update
sudo apt-get -y install grafana

sudo systemctl enable grafana-server
sudo systemctl start grafana-server
sudo systemctl status grafana-server
```

> The original reference used `apt-key` and `packages.grafana.com/oss/deb`. `apt-key` is deprecated, so the signed-by keyring method above is used instead. If you prefer the original approach, it still works on Ubuntu 22.04.

Open `http://<monitoring-ip>:3000`. Default login is `admin` / `admin` (you will be asked to change the password).

### 7.2 Add Prometheus as a data source

**Connections → Data sources → Add data source → Prometheus**

- Enable **Default**
- URL: `http://<monitoring-ip>:9090` (no trailing `/`)
- Click **Save & test**. A green tick confirms the connection.

### 7.3 Import dashboards

**Dashboards → New → Import**, enter the ID, click **Load**, select the **Prometheus** data source, then **Import**.

| Dashboard | ID | Source |
|---|---|---|
| Node Exporter Full | `1860` | https://grafana.com/grafana/dashboards/1860-node-exporter-full/ |
| Jenkins: Performance and Health Overview | `9964` | https://grafana.com/grafana/dashboards/9964-jenkins-performance-and-health-overview/ |

Save each dashboard after importing.

---

## Step 8: Argo CD Deployment

Run these on the `Ingress-Server` (where `kubectl` is configured for EKS).

### 8.1 Install Helm

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
helm version
```

### 8.2 Install Argo CD with Helm

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

kubectl create namespace argocd
helm install argocd argo/argo-cd --namespace argocd

kubectl get all -n argocd
```

### 8.3 Expose the Argo CD server

By default `argocd-server` is a `ClusterIP` service. Change it to `LoadBalancer`:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
kubectl get svc argocd-server -n argocd
```

Or fetch just the hostname with `jq`:

```bash
sudo apt install jq -y     # on Amazon Linux use: sudo yum install jq -y
kubectl get svc argocd-server -n argocd -o json | jq --raw-output '.status.loadBalancer.ingress[0].hostname'
```

### 8.4 Log in

- URL: the load balancer hostname above
- Username: `admin`
- Password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d
```

### 8.5 Create the Argo CD Application

In the Argo CD UI, click **New App** (or apply a manifest):

| Field | Value |
|---|---|
| Application Name | `microservices-app` |
| Project | `default` |
| Sync Policy | Automatic (enable **Prune** and **Self Heal**) |
| Repository URL | `https://github.com/iamajaypokharel/Microservices-ingress-ajay.git` |
| Revision | `master` |
| Path | folder containing your Kubernetes manifests (e.g. `k8s/`) |
| Cluster URL | `https://kubernetes.default.svc` |
| Namespace | the namespace your app deploys to |

Equivalent declarative manifest:

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: microservices-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/iamajaypokharel/Microservices-ingress-ajay.git
    targetRevision: master
    path: k8s            # change to your manifests folder
  destination:
    server: https://kubernetes.default.svc
    namespace: default   # change to your app namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

Click **Sync**. Argo CD pulls the manifests from GitHub and keeps the cluster in sync with Git. Any change you commit is automatically deployed.

---

## Verify the Application

```bash
kubectl get pods
kubectl get svc
kubectl get ingress

# External address of the ingress controller
kubectl get svc ingress-nginx-controller -n ingress-nginx
```

Open `http://<INGRESS_ELB_HOSTNAME>/` in the browser. Requests are routed by the Ingress rules to the matching microservice.

Check monitoring:

- Prometheus targets: `http://<monitoring-ip>:9090/targets`
- Grafana dashboards: `http://<monitoring-ip>:3000`
- Argo CD: application shows **Healthy** and **Synced**

---

## Cleanup

Delete resources to avoid AWS charges:

```bash
# Delete the app (via Argo CD UI or kubectl), then the ingress controller
kubectl delete -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/aws/deploy.yaml

# Remove the Argo CD LoadBalancer first so the ELB is released
helm uninstall argocd -n argocd

# Delete the EKS cluster
eksctl delete cluster --name kastro-cluster --region us-east-1
```

Then terminate the `Ingress-Server` and `Monitoring Server` EC2 instances.

---

## Troubleshooting

| Problem | Fix |
|---|---|
| Cannot reach Jenkins / Prometheus / Grafana | Check the security group ports; use `http://`, not `https://` |
| Jenkins: `permission denied` on Docker | `sudo usermod -aG docker jenkins` then restart Jenkins |
| Node Exporter target `0/1` | Open port `9100` on the Monitoring VM |
| Prometheus config will not load | Run `promtool check config /etc/prometheus/prometheus.yml` |
| Ingress has no external address | Wait a few minutes; check `kubectl get pods -n ingress-nginx` |
| Argo CD service has no hostname | Wait for the ELB to provision, then re-run `kubectl get svc argocd-server -n argocd` |
| `kubectl` cannot connect to EKS | `aws eks update-kubeconfig --name kastro-cluster --region us-east-1` |
| Ingress returns 404 | Verify the Ingress host/path rules and that the target Services have endpoints |

## Security Notes

- Never commit AWS keys, Docker Hub tokens or GitHub PATs to the repository. Keep them in Jenkins credentials.
- Prefer least-privilege IAM policies over `IAMFullAccess` and `eks:*` for anything beyond learning.
- Restrict security group sources instead of opening ports to the whole internet.
- Change the default Grafana password and the Argo CD initial admin password.
- Delete the PAT used in this guide once the project is done.

---

## Author

**Ajay Pokharel**
GitHub: [@iamajaypokharel](https://github.com/iamajaypokharel)

Based on the DevOps project by **Kastro Kiran V**.

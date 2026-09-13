```mermaid
graph TD
    User([User])

    subgraph Host Machine
        Ansible[Ansible Playbooks]
        Vagrant[Vagrant]
        Helm[Helm]
    end

    subgraph "Vagrant VMs (K3s Kubernetes Cluster)"
        direction TB
        CP(Control Plane Node)
        W1(Worker Node 1)
        W2(Worker Node 2)
        W3(Worker Node 3)
        W4(Worker Node 4)

        CP --- W1 & W2 & W3 & W4

        subgraph Cluster Services
            MetalLB[MetalLB LoadBalancer]
            Rancher[Rancher]
            KubeVirt[KubeVirt]
        end

        subgraph Workloads
            OTel[OpenTelemetry Demo]
            GitLab[GitLab CE with Google SSO]
        end
    end

    Vagrant -->|Provisions| CP
    Vagrant -->|Provisions| W1
    Vagrant -->|Provisions| W2
    Vagrant -->|Provisions| W3
    Vagrant -->|Provisions| W4

    Ansible -->|Deploys K3s & Services| CP

    Helm -->|Manually Deploys| OTel
    kubectl -->|Optionally Deploys| GitLab

    MetalLB -.->|Exposes Services| Rancher
    MetalLB -.->|Exposes Services| KubeVirt
    MetalLB -.->|Exposes Services| OTel
    MetalLB -.->|Exposes Services| GitLab

    User -->|Access via LoadBalancer IPs| MetalLB
```

The above diagram visualizes the high-level architecture of the microservices and Kubernetes deployment managed by this repository. It illustrates the infrastructure provisioned by Vagrant, configured via Ansible, and the core services running within the K3s cluster.

---

# Microservices & Kubernetes Deployment

This repository provisions a fully functional, multi-node K3s Kubernetes cluster on local Vagrant VMs using Ansible, and automatically deploys essential infrastructure tools like **MetalLB**, **Rancher**, and **KubeVirt**.

## Features

- **Automated K3s Cluster**: 1 Control Plane, 4 Worker Nodes.
- **MetalLB**: Provides external LoadBalancer IPs for services.
- **Rancher**: Web UI for multi-cluster management.
- **KubeVirt**: Run Virtual Machines natively alongside containers.
- **Optional GitLab CE**: Self-hosted DevOps platform with Google OAuth SSO and automatic user registration.

## Prerequisites

- [Vagrant](https://www.vagrantup.com/) & VirtualBox (or preferred provider)
- [Ansible](https://docs.ansible.com/)
- `kubectl` & `helm` installed on your host machine

---

## Getting Started

### 1. Bring up the Virtual Machines
This provisions the VMs as defined in the `Vagrantfile`.

```bash
vagrant up
```

### 2. Deploy the Cluster & Addons
The Ansible playbook will install K3s, configure the cluster, copy the kubeconfig to your host, and automatically install the addons (MetalLB, Rancher, and KubeVirt).

```bash
ansible-playbook -i inventory/inventory.yaml playbooks/deploy-k3s.yaml
```

*Note: The playbook is idempotent. You can safely re-run it if something fails.*

---

## Accessing the Applications

Once the playbook completes, wait a few minutes for all pods to become ready (`kubectl get pods -A -w`).

### Rancher
Rancher is exposed via a MetalLB LoadBalancer IP (usually `192.168.56.x`). Check the exact IP:
```bash
kubectl get svc rancher -n cattle-system
```
Access the IP in your browser via `https://<RANCHER_IP>`. (Default bootstrap password: `admin`).

### KubeVirt Manager
Manage your KubeVirt VMs from a web interface. Check the IP:
```bash
kubectl get svc kubevirt-manager -n kubevirt-manager
```
Access via `http://<KUBEVIRT_MANAGER_IP>`.

### GitLab (When Deployed)
If you deploy the optional GitLab workload, it is exposed via MetalLB LoadBalancer:
```bash
kubectl get svc gitlab-service -n gitlab
```
Access in your browser via your domain (`https://gitlab.madhoshyagnik.com`) or directly at `http://<GITLAB_LOADBALANCER_IP>`.

---

## Optional Workload Deployments

### 1. Manual Deployment: GitLab CE & GitLab Runner

Deploy an optional standalone GitLab CE instance configured with Google OAuth single sign-on, along with a native GitLab Runner using the Kubernetes executor for on-demand CI/CD pipelines:

1. **Configure Google OAuth Credentials**: Add your Google Client ID and Secret to [`kubernetes-manifests/gitlab/02-secret.yaml`](./kubernetes-manifests/gitlab/02-secret.yaml) (or provide via `kubectl create secret`).
2. **Deploy GitLab Server**:
   ```bash
   kubectl apply -k kubernetes-manifests/gitlab/
   ```
3. **Verify LoadBalancer IP**:
   ```bash
   kubectl get svc -n gitlab gitlab-service
   ```
4. **Bootstrap Admin Access**: Promote your user account to Administrator instantly with zero extra memory overhead:
   ```bash
   kubectl exec -it -n gitlab deployment/gitlab -c gitlab-ce -- \
     gitlab-psql -d gitlabhq_production -c "UPDATE users SET admin = true WHERE username = '<your_username>';"
   ```
5. **Deploy GitLab Runner (Optional CI/CD)**: Create an Instance Runner in GitLab Admin Area (`/admin/runners`), configure the runner token, and deploy:
   ```bash
   kubectl apply -k kubernetes-manifests/gitlab-runner/
   ```

> For the comprehensive guide, Google Cloud Console setup, DNS configuration, Runner setup, and troubleshooting, see the [GitLab Manual Deployment Guide](./Documentation/readme-manual-gitlab-deployment.md).

---

### 2. Manual Deployment: OpenTelemetry Demo

To deploy the OpenTelemetry Demo manually to your cluster, run the following Helm commands. 

> **Important**: To fix the known issue of the Prometheus pod crashing in Vagrant/k3s setups (due to local-path provisioner permissions with the nobody user), we disable the persistent volumes for Prometheus and Alertmanager in the helm install command below.

```bash
helm repo add open-telemetry https://open-telemetry.github.io/opentelemetry-helm-charts
helm repo update

helm install my-otel-demo open-telemetry/opentelemetry-demo \
  --namespace default \
  --set grafana.resources.limits.memory=512Mi \
  --set grafana.resources.requests.memory=256Mi \
  --set grafana.shadowBundledPlugins=true \
  --set prometheus.server.persistentVolume.enabled=false \
  --set prometheus.alertmanager.persistentVolume.enabled=false
```

Once running, you can expose the frontend via LoadBalancer:
```bash
kubectl patch svc frontend-proxy -p '{"spec":{"type":"LoadBalancer"}}'
kubectl get svc frontend-proxy -n default
```
Then access it via `http://<EXTERNAL_IP>:8080`.

---

## Cleanup

To completely remove K3s and all installed components from the VMs without destroying the VMs themselves:

```bash
ansible-playbook -i inventory/inventory.yaml playbooks/uninstall-k3s.yaml
```

If you want to destroy the VMs entirely:
```bash
vagrant destroy -f
```

## Architecture & Details

For deeper insights and manual step-by-step procedures, check out the [Documentation folder](./Documentation):
- [Setting Up a Local Multi-Node K3s Cluster](./Documentation/readme-manual-k3s-deployment.md)
- [Deploying Rancher Manager](./Documentation/readme-manual-rancher-deployment.md)
- [Deploying KubeVirt and virtctl](./Documentation/kubevirt-virtctl-manual-deployment.md)
- [Deploying GitLab CE and GitLab Runner](./Documentation/readme-manual-gitlab-deployment.md)
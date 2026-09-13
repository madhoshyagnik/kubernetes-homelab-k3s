# Manual Deployment: GitLab CE with Google OAuth SSO

This guide provides step-by-step instructions to manually deploy **GitLab Community Edition (CE)** on your local K3s Kubernetes homelab cluster, complete with **Google OAuth Single Sign-On (SSO)** and **automatic account creation on sign up**.

> [!NOTE]
> This deployment is **optional** and isolated in the `gitlab` namespace. It runs alongside your existing workloads without affecting cluster core services (MetalLB, Rancher, KubeVirt).

---

## Architecture Overview

```mermaid
flowchart TD
    User([User / Browser])
    Google[Google OAuth 2.0 API]

    subgraph "K3s Kubernetes Cluster"
        MetalLB[MetalLB LoadBalancer\n192.168.56.202:80, 22]
        
        subgraph "Namespace: gitlab"
            Service[Service: gitlab-service]
            Deploy[Deployment: gitlab-ce]
            PVC_Config[(PVC: gitlab-config\n2Gi)]
            PVC_Logs[(PVC: gitlab-logs\n5Gi)]
            PVC_Data[(PVC: gitlab-data\n15Gi)]
            Secret[Secret: gitlab-oauth-secret\nGoogle Client ID & Secret]
        end
    end

    User -->|1. Access Web UI /users/sign_in| MetalLB
    MetalLB --> Service --> Deploy
    User -->|2. Click 'Google' SSO| Google
    Google -->|3. Callback with Auth Code| User
    User -->|4. Redirect to /users/auth/google_oauth2/callback| MetalLB
    Deploy --> Secret
    Deploy --> PVC_Config & PVC_Logs & PVC_Data
```

### Key Design Decisions
- **Omnibus Container Deployment**: Deploys the official `gitlab/gitlab-ce` image as a single-pod Deployment backed by K3s `local-path` storage. This is far lighter and more stable for homelab environments than the multi-pod microservices Helm chart (which requires 8-16 GB RAM).
- **Environment-based OmniAuth Secrets**: Google OAuth credentials (`app_id` and `app_secret`) are injected from a Kubernetes `Secret` via `ENV` variables into `GITLAB_OMNIBUS_CONFIG`, keeping secrets out of version control.
- **MetalLB LoadBalancer Integration**: MetalLB automatically assigns an IP from the pool `192.168.56.200-192.168.56.220` to expose HTTP (port 80) and Git SSH (port 22).

---

## Prerequisites

1. **Functional K3s Cluster**: With MetalLB deployed and operational.
2. **Google Cloud Account**: Access to [Google Cloud Console](https://console.cloud.google.com/).
3. **Domain Name / Hostname**: A domain (e.g. `gitlab.madhoshyagnik.com`) or local hostname resolving to your MetalLB LoadBalancer IP.

---

## 1. Configure Google Cloud Console for OAuth 2.0

To enable Google sign-in on GitLab, create OAuth 2.0 credentials in Google Cloud Console:

### Step 1.1: Configure OAuth Consent Screen
1. Go to [Google Cloud Console > APIs & Services > OAuth consent screen](https://console.cloud.google.com/apis/credentials/consent).
2. Choose **External** user type and click **Create**.
3. Fill in required fields:
   - **App name**: `GitLab Homelab` (or your preferred name)
   - **User support email**: Your email address
   - **Developer contact information**: Your email address
4. Under **Authorized domains**, add your root domain (e.g. `madhoshyagnik.com`).
5. Click **Save and Continue** through Scopes (default `.../auth/userinfo.email` and `.../auth/userinfo.profile` are sufficient).

### Step 1.2: Create OAuth 2.0 Client ID
1. Navigate to **APIs & Services > Credentials**.
2. Click **+ CREATE CREDENTIALS** > **OAuth client ID**.
3. Set **Application type** to `Web application`.
4. Set **Name** to `GitLab Web Client`.
5. Under **Authorized JavaScript origins**, add:
   ```text
   https://gitlab.madhoshyagnik.com
   ```
6. Under **Authorized redirect URIs**, add the exact GitLab OmniAuth callback URI:
   ```text
   https://gitlab.madhoshyagnik.com/users/auth/google_oauth2/callback
   ```
   > [!IMPORTANT]
   > The callback URI must end exactly in `/users/auth/google_oauth2/callback`. Google OAuth requires `https://` for public domain names.

7. Click **Create**. Save the **Client ID** and **Client Secret**.

---

## 2. Configure Local DNS Resolution

Map the GitLab domain to your MetalLB LoadBalancer IP on your host machine (or your local DNS server / Pi-hole):

1. Check your MetalLB IP allocation pool (default: `192.168.56.200-192.168.56.220`).
2. Add the following entry to `/etc/hosts` on your host machine:
   ```text
   192.168.56.202 gitlab.madhoshyagnik.com
   ```
   *(Replace `192.168.56.202` with the actual IP assigned by MetalLB to `gitlab-service` once deployed).*

---

## 3. Configure Kubernetes Manifests

The deployment manifests are located in [`kubernetes-manifests/gitlab/`](../kubernetes-manifests/gitlab/).

### Step 3.1: Set Your Google OAuth Credentials
Edit [`kubernetes-manifests/gitlab/02-secret.yaml`](../kubernetes-manifests/gitlab/02-secret.yaml) (or create the secret directly with `kubectl`):

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: gitlab-oauth-secret
  namespace: gitlab
type: Opaque
stringData:
  GOOGLE_CLIENT_ID: "YOUR_GOOGLE_CLIENT_ID.apps.googleusercontent.com"
  GOOGLE_CLIENT_SECRET: "YOUR_GOOGLE_CLIENT_SECRET"
  GITLAB_ROOT_PASSWORD: "YourSecureRootPassword123!"
```

Alternatively, create it securely via CLI without editing the file:
```bash
kubectl create namespace gitlab --dry-run=client -o yaml | kubectl apply -f -

kubectl create secret generic gitlab-oauth-secret \
  --namespace gitlab \
  --from-literal=GOOGLE_CLIENT_ID="<YOUR_CLIENT_ID>" \
  --from-literal=GOOGLE_CLIENT_SECRET="<YOUR_CLIENT_SECRET>" \
  --from-literal=GITLAB_ROOT_PASSWORD="<YOUR_ROOT_PASSWORD>" \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Step 3.2: Review Omnibus Configuration
In [`kubernetes-manifests/gitlab/04-deployment.yaml`](../kubernetes-manifests/gitlab/04-deployment.yaml), the container uses `GITLAB_OMNIBUS_CONFIG`:

```ruby
external_url 'https://gitlab.madhoshyagnik.com'

# Web & Reverse Proxy settings
nginx['listen_https'] = false
nginx['listen_port'] = 80
gitlab_rails['gitlab_https'] = true

# Google OAuth SSO configuration
gitlab_rails['omniauth_enabled'] = true
gitlab_rails['omniauth_allow_single_sign_on'] = ['google_oauth2']
gitlab_rails['omniauth_block_auto_created_users'] = false
gitlab_rails['omniauth_providers'] = [
  {
    "name" => "google_oauth2",
    "app_id" => ENV['GOOGLE_CLIENT_ID'],
    "app_secret" => ENV['GOOGLE_CLIENT_SECRET'],
    "args" => { "access_type" => "offline", "approval_prompt" => "" }
  }
]

# Homelab resource tuning
puma['worker_processes'] = 2
sidekiq['concurrency'] = 10
prometheus_monitoring['enable'] = false
```

- `omniauth_allow_single_sign_on = ['google_oauth2']`: Allows users to sign in directly with Google without a pre-existing local account.
- `omniauth_block_auto_created_users = false`: Automatically activates newly registered accounts without administrator intervention.

---

## 4. Deploy GitLab

Apply all manifests with `kubectl`:

```bash
kubectl apply -k kubernetes-manifests/gitlab/
```

### Monitor Deployment Progress
GitLab Omnibus performs initial database migrations, secrets generation, and service startup on its first run (usually takes 3 to 5 minutes):

```bash
# Watch pod initialization
kubectl get pods -n gitlab -w

# Inspect container startup logs
kubectl logs -n gitlab deployment/gitlab -f
```

Wait until the pod reaches `1/1 Running` and readiness probe succeeds:
```text
NAME                      READY   STATUS    RESTARTS   AGE
gitlab-7f98bd46b5-x8q2z   1/1     Running   0          4m
```

---

## 5. Verify Service & Access GitLab

### Check Assigned LoadBalancer IP
```bash
kubectl get svc -n gitlab gitlab-service
```

Example output:
```text
NAME             TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)
gitlab-service   LoadBalancer   10.43.120.45   192.168.56.202   80:31234/TCP,22:30122/TCP
```

### Log In with Google
1. Open your browser and navigate to `https://gitlab.madhoshyagnik.space` (or `http://192.168.56.202` if testing via direct IP).
2. On the sign-in page, click the **Google** login button.
3. Sign in with your Google Account and grant permissions.
4. GitLab will authenticate you, automatically create your GitLab profile, and log you in immediately.

### Log In as Root (Administrator)
To access admin features:
- **Username**: `root`
- **Password**: The password defined in `GITLAB_ROOT_PASSWORD`, or retrieved from:
  ```bash
  kubectl exec -it -n gitlab deployment/gitlab -- cat /etc/gitlab/initial_root_password
  ```

---

## 6. Troubleshooting

### `redirect_uri_mismatch` Error
- **Cause**: The redirect URL sent by GitLab does not match the URL registered in Google Cloud Console.
- **Fix**: Verify that the Authorized Redirect URI in Google Cloud Console matches `external_url` + `/users/auth/google_oauth2/callback` exactly (including protocol `https://` and port if applicable).

### Pod CrashLoopBackOff or Out of Memory (OOMKilled)
- **Cause**: GitLab Omnibus requires at least 2.5 GB of free RAM.
- **Fix**: Check `kubectl describe pod -n gitlab` to see if the worker node ran out of memory. If necessary, adjust `puma['worker_processes'] = 1` or reduce worker thread concurrency in [`04-deployment.yaml`](../kubernetes-manifests/gitlab/04-deployment.yaml).

### SSL Certificate Warnings
- If you are terminating SSL at an external proxy or Ingress, ensure the proxy passes `X-Forwarded-Proto: https` so GitLab generates valid HTTPS links.

---

## 7. Cleanup

To remove the optional GitLab deployment and free up cluster resources:

```bash
kubectl delete -k kubernetes-manifests/gitlab/
```

To also delete persistent data volumes (warning: deletes all repositories and databases):
```bash
kubectl delete pvc --all -n gitlab
kubectl delete namespace gitlab
```

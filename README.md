# ArgoCD CI/CD Workshop Guide

This guide will walk you through setting up a complete CI/CD pipeline using k3s, ArgoCD, and GitOps principles. Follow each step in order to deploy your applications automatically.

## Prerequisites

- A Linux/macOS machine with internet access
- A GitHub account
- Basic knowledge of Kubernetes and Git

---

## Configuration Files

Most configuration files are available in the [`argocd` branch](https://github.com/cloudnativebd/cnbd-workshop-gitops/tree/argocd) of the repository. Click on any file to view or download it:

- `01-repository-secret.yaml` - Repository credentials for ArgoCD (created in Step 5 using a heredoc command)
- [`02-appproject.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/02-appproject.yaml) - ArgoCD Project definition with permissions and allowed repositories
- [`03-applicationset.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/03-applicationset.yaml) - ApplicationSet that uses list generator to create applications from specified folders
- [`04-webhook-secret.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/04-webhook-secret.yaml) - Webhook secret for GitHub webhook authentication

**Note:** You'll need to create the repository secret file in Step 5 and edit the other files with your repository information before applying them. See the steps below for details.

**Repository Structure Expected:**
The ApplicationSet expects your repository to have folders matching the `appname` values in `03-applicationset.yaml`. For example, if your ApplicationSet lists `frontend` and `backend`, your repository should have:
```
your-repo/
├── frontend/
│   └── (Kubernetes manifests)
└── backend/
    └── (Kubernetes manifests)
```

---

## Step 1: Install k3s

k3s is a lightweight Kubernetes distribution perfect for local development and workshops.

### 1.1 Install curl (if not already installed)

First, install the curl package for a smooth k3s installation.

**For Ubuntu/Debian:**
```bash
apt install curl -y
```

**For macOS:**
```bash
# curl is usually pre-installed, but if needed:
brew install curl
```

**For RHEL/CentOS/Fedora:**
```bash
yum install curl -y
# or for newer versions:
dnf install curl -y
```

### 1.2 Install k3s

Next, use the curl command to download and run the k3s installation script.

```bash
curl -sfL https://get.k3s.io | sh -
```

### 1.3 Verify k3s Installation

Once k3s has been installed, you can verify the k3s service using the following command.

```bash
systemctl status k3s
```

**Expected Output:**
```
● k3s.service - Lightweight Kubernetes
     Loaded: loaded (/etc/systemd/system/k3s.service; enabled; preset: enabled)
     Active: active (running) since Tue 2025-05-27 07:04:59 UTC; 2s ago
       Docs: https://k3s.io
    Process: 23962 ExecStartPre=/bin/sh -xc ! /usr/bin/systemctl is-enabled --quiet nm-cloud-setup.service 2>/dev/null (code=exited, status=0/SUCCESS)
    Process: 23964 ExecStartPre=/sbin/modprobe br_netfilter (code=exited, status=0/SUCCESS)
    Process: 23968 ExecStartPre=/sbin/modprobe overlay (code=exited, status=0/SUCCESS)
   Main PID: 23975 (k3s-server)
      Tasks: 20
     Memory: 464.0M (peak: 465.0M)
        CPU: 11.019s
     CGroup: /system.slice/k3s.service
             ├─23975 "/usr/local/bin/k3s server"
             └─23996 "containerd "
```

**Note:** On macOS, use `brew services list` to check k3s status instead of `systemctl`.

---

## Step 2: Access Kubernetes Cluster

You will need to configure kubectl to access and interact with the Kubernetes cluster via the command line.

### 2.1 Create kubectl Configuration Directory

First, create a directory to store kubectl configuration.

```bash
mkdir -p ~/.kube
```

### 2.2 Copy Kubernetes Configuration File

Next, copy the Kubernetes configuration file inside the .kube directory.

```bash
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chmod 600 ~/.kube/config
```

**Note:** On macOS, the path might be different. If the above doesn't work, try:
```bash
sudo cp /usr/local/etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chmod 600 ~/.kube/config
```

### 2.3 Set KUBECONFIG Environment Variable

Next, expose the path of the kubectl configuration file.

```bash
export KUBECONFIG=~/.kube/config
```

To make this permanent, add it to your shell configuration file:

**For bash:**
```bash
echo 'export KUBECONFIG=~/.kube/config' >> ~/.bashrc
source ~/.bashrc
```

**For zsh:**
```bash
echo 'export KUBECONFIG=~/.kube/config' >> ~/.zshrc
source ~/.zshrc
```

### 2.4 Verify Kubernetes Cluster Access

Next, verify the Kubernetes cluster nodes using the kubectl command.

```bash
kubectl get nodes
```

**Expected Output:**
```
NAME       STATUS   ROLES                  AGE   VERSION
ubuntu24   Ready    control-plane,master   51s   v1.32.5+k3s1
```

You can also see the Kubernetes cluster information using the following command.

```bash
kubectl cluster-info
```

**Expected Output:**
```
Kubernetes control plane is running at https://127.0.0.1:6443
CoreDNS is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
Metrics-server is running at https://127.0.0.1:6443/api/v1/namespaces/kube-system/services/https:metrics-server:https/proxy
```

---

## Step 3: Create Namespace for ArgoCD

Create a dedicated namespace for ArgoCD:

```bash
kubectl create namespace argocd
```

Verify the namespace was created:

```bash
kubectl get namespaces | grep argocd
```

---

## Step 4: Install ArgoCD

### 4.1 Install ArgoCD

Apply the official ArgoCD installation manifest:

```bash
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### 4.2 Wait for ArgoCD to be Ready

Wait for all ArgoCD pods to be running (this may take 2-3 minutes):

```bash
kubectl get pods -n argocd
```

All pods should show `Running` status.

### 4.3 Access ArgoCD UI

#### Option A: Port Forward

Forward the ArgoCD server port:

```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then access ArgoCD UI at: `https://localhost:8080` (accept the self-signed certificate warning)

#### Option B: NodePort (Recommended for Workshop)

If you prefer NodePort, patch the service:

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
```

Then get the port:

```bash
kubectl get svc argocd-server -n argocd
```

### 4.4 Get ArgoCD Admin Password

Retrieve the initial admin password:

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

**Note:** Save this password! You'll need it to login to the ArgoCD UI.
- Username: `admin`
- Password: (the output from the command above)

---

## Step 5: Configure Repository Secret

### 5.1 Generate SSH Key (If You Don't Have One)

If you don't have an SSH key, generate one:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

**Important:** Add the public key to your GitHub account:
- Copy your public key: `cat ~/.ssh/id_ed25519.pub`
- Go to GitHub → Settings → SSH and GPG keys → New SSH key
- Paste your public key and save

### 5.2 Create Repository Secret

You have two options for authentication:

**Option A: Use SSH (Recommended)**

Create the repository secret using a heredoc that automatically formats your SSH key:

```bash
cat <<EOF > 01-repository-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudnative-gitops-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: git@github.com:YOUR_USERNAME/YOUR_REPO.git
  sshPrivateKey: |
$(sed 's/^/    /' ~/.ssh/id_ed25519)
EOF
```

**⚠️ IMPORTANT:** Replace `YOUR_USERNAME/YOUR_REPO.git` with your actual repository URL.

**Note:** If your SSH key is in a different location or has a different name, update the path in the `sed` command. For example:
- If your key is at `/root/.ssh/id_ed25519`: `$(sed 's/^/    /' /root/.ssh/id_ed25519)`
- If your key is named `id_rsa`: `$(sed 's/^/    /' ~/.ssh/id_rsa)`

**Option B: Use HTTPS with Username/Password**

If you prefer HTTPS authentication, create the secret manually:

```bash
cat <<EOF > 01-repository-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: cloudnative-gitops-repo
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: repository
stringData:
  type: git
  url: https://github.com/YOUR_USERNAME/YOUR_REPO.git
  username: YOUR_GITHUB_USERNAME
  password: YOUR_GITHUB_PERSONAL_ACCESS_TOKEN
EOF
```

**⚠️ IMPORTANT:** Replace:
- `YOUR_USERNAME/YOUR_REPO.git` with your repository URL
- `YOUR_GITHUB_USERNAME` with your GitHub username
- `YOUR_GITHUB_PERSONAL_ACCESS_TOKEN` with your GitHub personal access token

### 5.3 Create GitHub Personal Access Token (If Using HTTPS)

If you're using HTTPS authentication (Option B), create a token:

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Click "Generate new token (classic)"
3. Give it a name (e.g., "ArgoCD Workshop")
4. Select the `repo` scope (for private repositories)
5. Click "Generate token"
6. **Copy the token immediately** (you won't see it again!)

### 5.4 Apply Repository Secret

```bash
kubectl apply -f 01-repository-secret.yaml
```

Verify the secret was created:

```bash
kubectl get secret cloudnative-gitops-repo -n argocd
```

---

## Step 6: Configure AppProject

### 6.1 Edit AppProject File

Open [`02-appproject.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/02-appproject.yaml) and make the following changes:

**⚠️ IMPORTANT CHANGES REQUIRED:**

1. **Repository URL** (Line 9): Replace `git@github.com:amdadulbari/cloudnative-gitops.git` with your repository URL:
   - If using SSH: `git@github.com:YOUR_USERNAME/YOUR_REPO.git`
   - If using HTTPS: `https://github.com/YOUR_USERNAME/YOUR_REPO.git`
   - **Important:** Use the same format (SSH or HTTPS) as you used in Step 5!

### 6.2 Apply AppProject

```bash
kubectl apply -f 02-appproject.yaml
```

**Note:** You can download the file from [here](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/02-appproject.yaml) if you haven't already.

Verify the AppProject was created:

```bash
kubectl get appproject -n argocd
```

---

## Step 7: Configure Webhook Secret (Optional but Recommended)

### 7.1 Generate Webhook Secret

Generate a secure random secret:

```bash
openssl rand -base64 32
```

**Save this secret!** You'll need it when configuring the GitHub webhook.

### 7.2 Edit Webhook Secret File

Open [`04-webhook-secret.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/04-webhook-secret.yaml) and replace the existing secret value on **Line 18** with the secret you just generated.

**Current value:** `lv4Np5cE+oh0Kp7PvKpHUypF0VXy7qVjyRpD5sxPtF0=`

**Replace with:** Your generated secret (the output from the command above)

### 7.3 Apply Webhook Secret

```bash
kubectl apply -f 04-webhook-secret.yaml
```

**Note:** You can download the file from [here](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/04-webhook-secret.yaml) if you haven't already.

**Note:** After applying all manifests, you'll configure the GitHub webhook using this secret. See `webhook-setup.md` for detailed instructions.

---

## Step 8: Configure ApplicationSet

### 8.1 Edit ApplicationSet File

Open [`03-applicationset.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/03-applicationset.yaml) and make the following changes:

**⚠️ IMPORTANT CHANGES REQUIRED:**

1. **Repository URL** (Line 21): Replace `git@github.com:amdadulbari/cloudnative-gitops.git` with your repository URL:
   - If using SSH: `git@github.com:YOUR_USERNAME/YOUR_REPO.git`
   - If using HTTPS: `https://github.com/YOUR_USERNAME/YOUR_REPO.git`
   - **Important:** Use the same format (SSH or HTTPS) as you used in Steps 5 and 6!

2. **Target Branch** (Line 22): Replace `main` with your repository's default branch if different:
   - Common options: `main`, `master`, `develop`, or a specific branch name

3. **Application Names** (Lines 10-11): Update the `elements` list to match your repository structure. Each `appname` should correspond to a folder in your repository. For example:
   - If your repo has `backend`, `frontend`, and `api` folders:
     ```yaml
     elements:
       - appname: backend
       - appname: frontend
       - appname: api
     ```
   - If your repo has different folder names, update accordingly:
     ```yaml
     elements:
       - appname: service-a
       - appname: service-b
     ```

4. **Target Namespace** (Line 26): Update the `namespace` field if you want to deploy to a different namespace:
   - Current value: `workshop`
   - You can change it to any namespace name, e.g., `my-apps`, `production`, `staging`, etc.
   - **Note:** The namespace will be created automatically if it doesn't exist (due to `CreateNamespace=true` in syncOptions)

### 8.2 Apply ApplicationSet

```bash
kubectl apply -f 03-applicationset.yaml
```

**Note:** You can download the file from [here](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/03-applicationset.yaml) if you haven't already.

---

## Step 9: Verify Setup

### 9.1 Check ApplicationSet Status

```bash
kubectl get applicationset -n argocd
```

The ApplicationSet should show `Ready` status.

### 9.2 Check Created Applications

```bash
kubectl get applications -n argocd
```

You should see one Application for each folder you specified in the ApplicationSet.

### 9.3 View Application Details

```bash
kubectl describe applicationset cloudnative-gitops-apps -n argocd
```

### 9.4 Check Application Sync Status

```bash
kubectl get applications -n argocd -o wide
```

Applications should show `Synced` and `Healthy` status once they've been deployed.

---

## Step 10: Access ArgoCD UI and Verify

### 10.1 Login to ArgoCD UI

1. Open your browser and go to `https://localhost:8080` (or your NodePort URL)
2. Accept the self-signed certificate warning
3. Login with:
   - Username: `admin`
   - Password: (the password from Step 4.4)

### 10.2 Verify Applications

In the ArgoCD UI, you should see:
- Your ApplicationSet listed
- Individual Applications for each service folder
- Applications syncing and deploying to your cluster

---

## Step 11: Test CI/CD Pipeline

### 11.1 Make a Change to Your Repository

1. Make a change to any Kubernetes manifest in one of your service folders
2. Commit and push the change:

```bash
git add .
git commit -m "Test CI/CD pipeline"
git push
```

### 11.2 Watch ArgoCD Sync

In the ArgoCD UI, you should see:
- The application automatically detecting the change
- The application syncing the new changes
- The changes being deployed to your cluster

### 11.3 Verify Deployment

Check that your changes were deployed:

```bash
kubectl get pods -n <your-namespace>
```

---

## Quick Reference: All Changes Required

Here's a summary of all the places you need to update for your repository:

| File | What to Change | Location | Current Value |
|------|---------------|----------|--------------|
| `01-repository-secret.yaml` (created in Step 5) | Repository URL (SSH format) | In heredoc command | `git@github.com:YOUR_USERNAME/YOUR_REPO.git` |
| `01-repository-secret.yaml` (created in Step 5) | SSH Private Key Path | In heredoc command | `~/.ssh/id_ed25519` (update if different) |
| [`02-appproject.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/02-appproject.yaml) | Repository URL | Line 9 | `git@github.com:amdadulbari/cloudnative-gitops.git` |
| [`03-applicationset.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/03-applicationset.yaml) | Repository URL | Line 21 | `git@github.com:amdadulbari/cloudnative-gitops.git` |
| [`03-applicationset.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/03-applicationset.yaml) | Target Branch | Line 22 | `main` |
| [`03-applicationset.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/03-applicationset.yaml) | Application Names (service folders) | Lines 10-11 | `frontend`, `backend` |
| [`03-applicationset.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/03-applicationset.yaml) | Target Namespace | Line 26 | `workshop` |
| [`04-webhook-secret.yaml`](https://github.com/cloudnativebd/cnbd-workshop-gitops/blob/argocd/04-webhook-secret.yaml) | Webhook secret | Line 18 | `lv4Np5cE+oh0Kp7PvKpHUypF0VXy7qVjyRpD5sxPtF0=` |

**Important Notes:**
- Use the same repository URL format (SSH or HTTPS) in all three files (01, 02, and 03)
- The application names in `03-applicationset.yaml` must match the folder names in your repository
- Generate a new webhook secret for security (don't use the example value)

---

## Troubleshooting

### Applications Not Created

1. Check ApplicationSet status:
   ```bash
   kubectl describe applicationset cloudnative-gitops-apps -n argocd
   ```

2. Check ApplicationSet controller logs:
   ```bash
   kubectl logs -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller
   ```

3. Verify repository access:
   ```bash
   kubectl get secret cloudnative-gitops-repo -n argocd -o yaml
   ```

### Repository Access Issues

- For private repos, ensure credentials are correct in `01-repository-secret.yaml`
- If using SSH: Verify your SSH key path is correct in the heredoc command and the key is added to GitHub
- If using HTTPS: Verify your GitHub token has the `repo` scope
- Ensure the repository URL format (SSH or HTTPS) is consistent across all configuration files (02-appproject.yaml and 03-applicationset.yaml)
- Check if the repository URL is accessible from your cluster

### Applications Not Syncing

1. Check application status in ArgoCD UI
2. Verify the `appname` values in `03-applicationset.yaml` match the actual folder names in your repository
3. Verify the path in your repository contains valid Kubernetes manifests
4. Check if the target namespace exists or if `CreateNamespace=true` is set (it should be)
5. Verify the `targetRevision` (branch name) in `03-applicationset.yaml` matches your repository's branch

### ArgoCD Server Not Accessible

1. Check if port-forward is still running
2. Verify ArgoCD pods are running:
   ```bash
   kubectl get pods -n argocd
   ```
3. Restart port-forward if needed:
   ```bash
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```

---

## Additional Resources

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [ApplicationSet Documentation](https://argo-cd.readthedocs.io/en/stable/operator-manual/applicationset/)
- [k3s Documentation](https://k3s.io/)
- [GitHub Webhook Setup Guide](./webhook-setup.md)

---

## Workshop Checklist

Use this checklist to ensure you've completed all steps:

- [ ] Step 1: k3s installed and running
- [ ] Step 2: kubectl configured and cluster accessible
- [ ] Step 3: `argocd` namespace created
- [ ] Step 4: ArgoCD installed and accessible
- [ ] Step 5: Repository secret created using heredoc command with your repo URL and SSH key
- [ ] Step 6: AppProject configured with your repo URL
- [ ] Step 7: Webhook secret generated and configured
- [ ] Step 8: ApplicationSet configured with your repo URL and service folders
- [ ] Step 9: Applications created and visible in `kubectl get applications`
- [ ] Step 10: ArgoCD UI accessible and applications visible
- [ ] Step 11: Tested CI/CD by pushing a change and verifying sync

---

**Congratulations!** You've successfully set up a GitOps CI/CD pipeline with ArgoCD!

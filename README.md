### 1: Homework Assignment 1: Metrics stack and a dashboard
Install kube-prometheus-stack into a monitoring
<img width="975" height="155" alt="image" src="https://github.com/user-attachments/assets/b386a58a-6b27-4254-a90c-04bb56a28ae0" />


1.	Show that the pods are running and that the PVCs are Bound

<img width="975" height="155" alt="image" src="https://github.com/user-attachments/assets/ee142437-9494-43e9-b65f-5441fd55199b" />



Open the Prometheus UI (port-forward is enough) and look at Status → Targets. Report how many targets there are and how many are up.

<img width="975" height="482" alt="image" src="https://github.com/user-attachments/assets/42408eed-2099-41a0-ad9b-fc053eabf6de" />


•	Problem: The `kube-proxy` target (or similar master components) appears with a status of `DOWN`, accompanied by a "Connection refused" error.
•	Cause: Within the component's ConfigMap (e.g., as seen via `kubectl get cm kube-proxy -n kube-system -o yaml`), the `metricsBindAddress` parameter is set to `127.0.0.1:10249` by default. This restricts the service to listening for metrics only on the node's local interface. Since Prometheus resides within an isolated pod network, it is physically unable to reach this address. This is a default Kubernetes security behavior, not a cluster malfunction.

<img width="975" height="781" alt="image" src="https://github.com/user-attachments/assets/223e5a5e-c5cc-4d19-bb18-fc3fd3f2b262" />



4	Open Grafana, make sure Prometheus is wired as a data source.

<img width="975" height="272" alt="image" src="https://github.com/user-attachments/assets/ab4031fc-5989-4c6c-8f92-eb8e06c63535" />


5	Build a dashboard with two panels per pod: CPU and memory. Useful starting points:
•	Useful starting points:
•	rate(container_cpu_usage_seconds_total{namespace="<your-ns>"}[5m])
•	container_memory_working_set_bytes{namespace="<your-ns>"}
•	Add a dashboard variable for the node and a panel showing the number of pods per node:
•	sum(kube_pod_info{node=~"$node"}) by (node)

<img width="975" height="434" alt="image" src="https://github.com/user-attachments/assets/56885f0c-33bd-42f2-9380-66d5c09a5331" />


Deliverables: your values file, the dashboard exported as JSON, and a screenshot of the dashboard with data in it.

Test_dashbord-1792007652064.json

<img width="975" height="434" alt="image" src="https://github.com/user-attachments/assets/9c0a3f14-6348-4c53-857d-0edd918a40b1" />


Homework Assignment 2: Alerting to Slack




# Homework Assignment 1. ArgoCD Deployment and Application

This repository contains the declarative configuration manifests for deploying an infrastructure stack using **ArgoCD** and managing encrypted secrets securely via GitOps with **Bitnami Sealed Secrets**.

The project is structured according to the **App-of-Apps (Pattern of Patterns)** design pattern.

---

## 🛠 Project Structure

```text
.
├── argo-apps/              # ArgoCD App-of-Apps management manifests
│   ├── project.yaml        # AppProject defining environments and permissions
│   ├── app-jenkins.yaml    # Application definition for Jenkins (Helm source)
│   └── app-secrets.yaml    # Application definition for Sealed Secrets (Git source)
├── apps-data/              # Sealed Secrets objects (Safe to store in Git)
│   └── git-secret-sealed.yaml
├── argo-core/              # Core cluster components installation manifests
│   └── argo-install.yaml
├── root-app.yaml           # Root bootstrapping Application definition
└── README.md
```

---


### Step 1. Deploy ArgoCD into the Cluster

1. Create the `argocd` namespace and deploy the core ArgoCD components:

```
  kubectl create namespace argocd
  kubectl apply -n argocd --server-side --force-conflicts -f argo-install.yaml
```

<img width="975" height="265" alt="image" src="https://github.com/user-attachments/assets/58843c6b-6d00-4c4b-9173-b473db7614a2" />

Checking web-interface  argocd.k8s-7.sa
```
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 –d   
```
<img width="975" height="556" alt="image" src="https://github.com/user-attachments/assets/111d7ebf-9c48-45d0-a5a9-2dc593039c75" />

---

### Step 2. Handle Git Secrets as SealedSecret Objects

To safely store sensitive credentials in a public Git repository, raw Kubernetes Secrets are encrypted locally using the controller's public key certificate.

1. Fetch the master public key certificate from the active cluster controller:
   ```bash
   kubectl get secret -n kube-system sealed-secrets-key2bglq -o jsonpath='{.data.tls\.crt}' | base64 -d > pub-cert.pem
   ```

2. Generate a local plaintext secret template (do **NOT** commit this file):
   ```bash
   kubectl create secret generic my-git-secret \
     --from-literal=username=myuser \
     --from-literal=password=mypassword \
     --dry-run=client -o yaml > secret-plain.yaml
   ```

3. Encrypt the secret using `kubeseal` and place it in the application resources directory:
   ```bash
   kubeseal --cert pub-cert.pem < secret-plain.yaml > apps-data/git-secret-sealed.yaml
   ```

---

### Step 3. Deploy Jenkins

* Used the previous repository as the source for the Helm chart - 
<img width="1031" height="217" alt="image" src="https://github.com/user-attachments/assets/c8977040-6970-4cc4-b9b0-036840e99d45" />

```
# apply the project so that ArgoCD knows the security rules.
kubectl apply -f argo-apps/my-project.yaml

# Registering a Helm repository in ArgoCD
kubectl apply -f argo-apps/helm-repo.yaml

# Deploying the Jenkins application via a Helm chart
kubectl apply -f argo-apps/app-deployment.yaml
kubectl apply -f secrets-app.yaml

  ├── project.yaml        # AppProject defining environments and permissions
  ├── app-jenkins.yaml    # Application definition for Jenkins (Helm source)
  └── app-secrets.yaml    # Application definition for Sealed Secrets (Git source)
```

<img width="975" height="479" alt="image" src="https://github.com/user-attachments/assets/2e8e4d86-9124-46f2-9382-0b142abf4c0e" />
<img width="975" height="553" alt="image" src="https://github.com/user-attachments/assets/45138fc5-b022-4ba9-b444-43291ee844db" />


### Step 4. Refactored the structure to the App-of-Apps
The App-of-Apps pattern is used to fully automate the GitOps process: instead of manually running `kubectl apply` commands for dozens of manifests, you deploy just a single root application.

The root application monitors the `argo-apps/` folder and dynamically provisions the defined applications.

<img width="1846" height="692" alt="image" src="https://github.com/user-attachments/assets/c1e7657d-d0b9-4f40-ba57-95debd47488b" />

<img width="975" height="509" alt="image" src="https://github.com/user-attachments/assets/c2301a65-5943-4530-94ae-349b36a354ea" />


### 1: Homework Assignment 1: Metrics stack and a dashboard
Install kube-prometheus-stack into a monitoring
<img width="975" height="155" alt="image" src="https://github.com/user-attachments/assets/b386a58a-6b27-4254-a90c-04bb56a28ae0" />


*	Show that the pods are running and that the PVCs are Bound

<img width="975" height="179" alt="image" src="https://github.com/user-attachments/assets/82fb43e5-dcb3-47ba-ba1e-4fe625211add" />




* Open the Prometheus UI (port-forward is enough) and look at Status → Targets. Report how many targets there are and how many are up.

<img width="975" height="482" alt="image" src="https://github.com/user-attachments/assets/42408eed-2099-41a0-ad9b-fc053eabf6de" />


•	Problem: The `kube-proxy` target (or similar master components) appears with a status of `DOWN`, accompanied by a "Connection refused" error.
•	Cause: Within the component's ConfigMap (e.g., as seen via `kubectl get cm kube-proxy -n kube-system -o yaml`), the `metricsBindAddress` parameter is set to `127.0.0.1:10249` by default. This restricts the service to listening for metrics only on the node's local interface. Since Prometheus resides within an isolated pod network, it is physically unable to reach this address. This is a default Kubernetes security behavior, not a cluster malfunction.

<img width="975" height="781" alt="image" src="https://github.com/user-attachments/assets/223e5a5e-c5cc-4d19-bb18-fc3fd3f2b262" />



*	Open Grafana, make sure Prometheus is wired as a data source.

<img width="975" height="272" alt="image" src="https://github.com/user-attachments/assets/ab4031fc-5989-4c6c-8f92-eb8e06c63535" />


*	Build a dashboard with two panels per pod: CPU and memory. Useful starting points:
•	Useful starting points:
•	rate(container_cpu_usage_seconds_total{namespace="<your-ns>"}[5m])
•	container_memory_working_set_bytes{namespace="<your-ns>"}
•	Add a dashboard variable for the node and a panel showing the number of pods per node:
•	sum(kube_pod_info{node=~"$node"}) by (node)

<img width="975" height="434" alt="image" src="https://github.com/user-attachments/assets/56885f0c-33bd-42f2-9380-66d5c09a5331" />


* Deliverables: your values file, the dashboard exported as JSON, and a screenshot of the dashboard with data in it.

Test_dashbord-1792007652064.json




### Homework Assignment 2: Alerting to Slack

* Enabling Alertmanager
```
# Check the release status and the appearance of pods:
kubectl get hr kps -n monitoring
kubectl get pods -n monitoring
```
<img width="975" height="320" alt="image" src="https://github.com/user-attachments/assets/241d236a-7a53-4316-be31-3fafca90e627" />
The screenshot shows that the pod `alertmanager-kps-kube-prometheus-stack-alertmanager-0` has started successfully and is in the `Running` state.

* Creating a Slack Webhook
We use an official, documented chart parameter for integrating external secrets — alertmanager.alertmanagerSpec.configSecret.

* Create a local configuration secret directly within the cluster:
```
kubectl create secret generic alertmanager-kps-generated -n monitoring --from-literal=alertmanager.yaml='global:
  resolve_timeout: 5m
route:
  group_by: ["alertname", "namespace", "pod"]
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 1h
  receiver: "slack-notifications"
receivers:
  - name: "slack-notifications"
    slack_configs:
      - channel: "#all-test"
        api_url: "https://slack.com"
        send_resolved: true
        title: "[{{ .Status | toUpper }}] Мониторинг Кластера"
        text: "{{ range .Alerts }}{{ .Annotations.summary }}\n{{ .Annotations.description }}\n{{ end }}"'
```
* Edited monitoring/kps.yaml and sent a test message from the terminal
<img width="975" height="274" alt="image" src="https://github.com/user-attachments/assets/364acfaa-f2ae-4806-a4ed-6788cfa9f5f2" />

**Launching a high CPU load simulation**
For our `HighPodCpuLoad` alert rule to trigger, we need to force the CPU of a pod to continuously consume more than 0.8 cores for 2 minutes.
To do this, we will launch a temporary load-generating pod in the cluster:

```
kubectl run load-generator -n monitoring --rm -i --tty --image=ubuntu -- bash -c "apt-get update && apt-get install -y stress && stress --cpu 2"
...
Preparing to unpack .../stress_1.0.7-2_amd64.deb ...
Unpacking stress (1.0.7-2) ...
Setting up stress (1.0.7-2) ...
stress: info: [1] dispatching hogs: 2 cpu, 0 io, 0 vm, 0 hdd
```
<img width="975" height="86" alt="image" src="https://github.com/user-attachments/assets/34b21dc7-bdc9-48cd-8161-cd401d100c02" />


<img width="975" height="530" alt="image" src="https://github.com/user-attachments/assets/69984bdb-8d0b-4982-b0cf-391120e37eab" />

Viewing the status of an alert in Prometheus: 
HighPodCpuLoad
<img width="975" height="434" alt="image" src="https://github.com/user-attachments/assets/a6e68bdb-3671-4905-8464-a6d7f43b1d56" />

* Alert successfully delivered:
<img width="975" height="681" alt="image" src="https://github.com/user-attachments/assets/811fb9af-8f8c-48f2-8f2b-59699e275bc1" />


**Explain in your README what for: does in an alert rule and why an alert without it is a bad idea.**

•	What is the purpose of the `for:` parameter in an alert rule?
The `for: 2m` parameter defines a time buffer (duration). It specifies that the metric threshold must be continuously exceeded for 2 minutes before the alert transitions from the `PENDING` state to the `FIRING` state and sends a notification.

•	Why is an alert without a `for` clause a bad idea?
Without this parameter, an alert will trigger instantly. Any momentary spike in load—such as during application startup or garbage collection—will result in a flood of spam in Slack. This leads to "alert fatigue": engineers become accustomed to constant, false notifications and eventually start ignoring the channel, which causes them to miss genuine, critical incidents.



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


test push for 20.Jenkins!!!

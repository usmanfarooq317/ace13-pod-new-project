# ACE Kubernetes Deployment Project

This project deploys an **IBM App Connect Enterprise (ACE)** application on Kubernetes.
It features **Persistent Storage (PVC)** for BAR files, **Horizontal Pod Autoscaling (HPA)**, and a complete **Jenkins Automation Pipeline**.

## Project Structure

- **`deployment.yaml`**: Main application deployment. Configures ACE to mount the PVC and deploy the BAR file on startup.
- **`hpa.yaml`**: Autoscaling configuration (Min 2 pods, Max 5 pods, Target CPU 50%).
- **`pvc.yaml`**: PersistentVolumeClaim (1Gi) to store the BAR file.
- **`pvc-loader.yaml`**: Temporary pod used to upload the BAR file to the PVC.
- **`Jenkinsfile`**: CI/CD pipeline script to automate the entire process.
- **`BVSRegFix2.bar`**: The ACE BAR file application to be deployed.

---

## Prerequisites

1.  **Minikube** (or any Kubernetes cluster)
2.  **kubectl** configured to talk to your cluster.
3.  **Jenkins** (for automation) with the *Kubernetes CLI* plugin installed.
4.  **Git** installed.

---

## Setup & Deployment

You can deploy this project either **Manually** or via **Jenkins Automation**.

### Option 1: Jenkins Automation (Recommended)

1.  **Configure Git Repository**:
    -   Ensure this project is pushed to your Git repository (e.g., `https://github.com/usmanfarooq317/ace13-pod-new-project`).

2.  **Add Kubernetes Credentials**:
    -   Get your kubeconfig: `kubectl config view --flatten --minify > kubeconfig.yaml`
    -   In Jenkins: **Manage Jenkins** -> **Credentials** -> **Global** -> **Add Credentials**.
    -   **Kind**: Secret file.
    -   **File**: Upload `kubeconfig.yaml`.
    -   **ID**: `kubeconfig-credentials-id`.

3.  **Create Pipeline Job**:
    -   Create a new Pipeline job named `ace13-pod-project`.
    -   **Pipeline Section**: Select "Pipeline script from SCM".
    -   **SCM**: Git.
    -   **Repository URL**: Your Repo URL.
    -   **Script Path**: `Jenkinsfile`.

4.  **Run**: Click **Build Now**. Check the Console Output to see the logs streaming from the pods.

---

### Option 2: Manual Deployment

If you want to run this locally without Jenkins, follow these steps:

#### 1. Initialize Storage
Create the Persistent Volume and the loader pod:
```bash
kubectl apply -f pvc.yaml
kubectl apply -f pvc-loader.yaml
```
Wait for the loader pod to be ready:
```bash
kubectl wait --for=condition=Ready pod/pvc-loader --timeout=60s
```

#### 2. Upload BAR File
Copy the local BAR file into the cluster storage:
```bash
kubectl cp BVSRegFix2.bar pvc-loader:/mnt/data/BVSRegFix2.bar
```
Clean up the loader pod:
```bash
kubectl delete pod pvc-loader --force --grace-period=0
```

#### 3. Deploy Application
Apply the main deployment and autoscaler:
```bash
kubectl apply -f deployment.yaml
kubectl apply -f hpa.yaml
```

---

## Verification

### Check Pod Status
Ensure pods are running (you desire 2 replicas):
```bash
kubectl get pods
# STATUS should be Running
```

### Verify Broker and Server
Exec into a pod to check the status of the Integration Node (Broker) and Server:
```bash
# Replace [pod-name] with actual pod name
kubectl exec [pod-name] -- bash -c "source /opt/ibm/ace-*/server/bin/mqsiprofile && mqsilist"
```

### Verify BAR Deployment
Check if the specific BAR file (`BVSRegFix2`) application is deployed:
```bash
kubectl exec [pod-name] -- bash -c "source /opt/ibm/ace-*/server/bin/mqsiprofile && mqsilist PROD -e prodserver -d 2"
```
**Expected Output**:
> BIP1877I: REST API 'TMFBService' on integration server 'prodserver' is running.
> Deployed: ... in Bar file '/home/aceuser/bars/BVSRegFix2.bar'

---

## Troubleshooting

- **`ContainerCreating` Hangs**: Check `kubectl describe pod [pod-name]`. If it's a `MountVolume` error, restart Minikube (`minikube stop && minikube start`).
- **TLS Handshake Timeout**: Restart Minikube.

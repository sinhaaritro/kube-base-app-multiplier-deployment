Kubernetes setup for [kube-base-app-multiplier](https://github.com/sinhaaritro/kube-base-app-multiplier)

## **Secretes Used** 
1. **GH_PAT**: This is in Repo. But will need the code from `GHA_IMAGE_PUSHER` secret at the user level
2. **k8s-image-pull-token**: This is at the user level


## **Steps**

#### **Phase 1: First-Time Setup (Only do this once or if your cluster is down)**

**Step 1.1: Create the GitHub Personal Access Token (PAT)**

You need a PAT to allow your Kubernetes cluster to download your private container image from the GitHub Container Registry (GHCR).

1.  **On the GitHub Website:**
    *   Go to **Settings** > **Developer settings** > **Personal access tokens** > **Tokens (classic)**.
    *   Click **Generate new token (classic)**.
    *   Give it a **Note** (e.g., `K8S_PULL_SECRET`).
    *   Set an **Expiration**.
    *   Check the scopes: **`repo`** and **`write:packages`**.
    *   Click **Generate token**.
2.  **CRITICAL:** Copy the new token (it starts with `ghp_...`) and paste it into a temporary notepad. **You will only see this token once.**

**Step 1.2 (Optional): Create the `kind` Cluster**

Check if your cluster is already running. If not, create it.

1.  **Check for an existing cluster:**
    ```bash
    kind get clusters
    ```
    *   If you see your cluster name listed, you can skip to Phase 2.
    *   If it says `No kind clusters found`, proceed to the next step.

2.  **Create a simple cluster (no config file):**
    ```bash
    KIND_EXPERIMENTAL_PROVIDER=podman kind create cluster
    ```
    This will take a minute or two. When it's done, your cluster will be running, and `kubectl` will be automatically configured to use it.

---

#### **Phase 2: Deploying the Application**

**Step 2.1: Clone the Deployment Repository**

Get the Kubernetes configuration files onto your VM.

1.  Navigate to your home directory:
    ```bash
    cd ~
    ```
2.  Clone your deployment repository:
    ```bash
    git clone https://github.com/sinhaaritro/kube-base-app-multiplier-deployment.git
    ```
3.  Navigate into the new directory:
    ```bash
    cd kube-base-app-multiplier-deployment
    ```

**Step 2.2: Create the Kubernetes Image Pull Secret**

Give your Kubernetes cluster the PAT you created so it can log in to GHCR.

1.  Run the following `kubectl` command.
    *   Replace `YOUR_PERSONAL_ACCESS_TOKEN` with the actual token you copied from GitHub.

    ```bash
    kubectl create secret docker-registry ghcr-creds \
      --docker-server=ghcr.io \
      --docker-username=sinhaaritro \
      --docker-password=YOUR_PERSONAL_ACCESS_TOKEN
    ```
    You should see the output `secret/ghcr-creds created`.

**Step 2.3: Modify the `deployment.yaml` to Use the Secret**

Now, tell your application's deployment to use the secret you just made.

1.  Open `deployment.yaml` with a text editor:
    ```bash
    nano deployment.yaml
    ```
2.  Add the `imagePullSecrets` section directly under the `spec:` line, as shown below:

    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
      name: multiplier-app-deployment
    spec:
      # --- ADD THIS SECTION ---
      imagePullSecrets:
      - name: ghcr-creds
      # ------------------------
      replicas: 1
      selector:
        matchLabels:
          app: multiplier-app
    # ... rest of the file is the same ...
    ```

3.  Save the file and exit `nano` by pressing `Ctrl+X`, then `Y`, then `Enter`.

**Step 2.4: Apply the Kubernetes Configuration**

Deploy your application to the cluster.

1.  Run the apply command from inside the `kube-base-app-multiplier-deployment` directory:
    ```bash
    kubectl apply -f .
    ```
    You should see `deployment.apps/multiplier-app-deployment configured` and `service/multiplier-app-service created` (or unchanged).

**Step 2.5: Verify the Deployment**

Check the status of your pod.

1.  Watch the pod status:
    ```bash
    kubectl get pods
    ```
    You might briefly see `ContainerCreating` or `ErrImagePull` on a new pod. Within a minute, it should change to `Running`. If an old pod is stuck in `ErrImagePull`, it will be terminated and replaced by a new one that works.

2.  **Desired Output:**
    ```
    NAME                                         READY   STATUS    RESTARTS   AGE
    multiplier-app-deployment-xxxxxxxxxx-yyyyy   1/1     Running   0          30s
    ```

---

#### **Phase 3: Accessing Your Running Application**

Since we created the cluster without a config file, we must use `port-forward` to access the application.

1.  **Start the port forward:** This command will take over your current terminal window.
    ```bash
    # This forwards port 8888 on your VM to port 8000 inside the running pod.
    kubectl port-forward deployment/multiplier-app-deployment 8888:8000
    ```
    You will see output like `Forwarding from 127.0.0.1:8888 -> 8000`.

2.  **Access the application:** Open a web browser on your local machine and go to:
    **`http://localhost:8888`**
    (This works if your VS Code is forwarding ports from the VM. If not, you would use `curl http://localhost:8888` in a *new* VM terminal).

You have now successfully deployed your containerized application to Kubernetes

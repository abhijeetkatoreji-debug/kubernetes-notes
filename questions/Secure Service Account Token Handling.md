### **Kubernetes Practice Lab: Secure Service Account Token Handling**

Here is the complete lab environment, including the scenario, a setup script to prepare your cluster, the step-by-step solution, and the official documentation links.

---

### **1. The Question (Exam Scenario)**

**Context:**
A security audit has identified a Deployment improperly handling service account tokens, which could lead to security vulnerabilities.

**Task:**

1. Modify the existing Service Account `stats-monitor-sa` in the namespace `monitoring` to turn off automounting of API credentials.
2. Modify the existing Deployment `stats-monitor` in the namespace `monitoring` to inject a ServiceAccount token.
3. Use a Projected Volume named `token` to inject the ServiceAccount token and ensure that it is mounted **read-only** at the path `/var/run/secrets/kubernetes.io/serviceaccount`. *(The token file itself should be available at `/var/run/secrets/kubernetes.io/serviceaccount/token`)*.
4. The Deployment's manifest file can be found at `~/stats-monitor/deployment.yaml`. Apply your changes using this file.

---

### **2. Lab Setup (Run this before trying the solution)**

Run this script to create the insecure starting state described in the prompt.

```bash
# 1. Create the namespace
kubectl create ns monitoring

# 2. Create the ServiceAccount (automounting is true by default)
kubectl create sa stats-monitor-sa -n monitoring

# 3. Create the directory for the manifest
mkdir -p ~/stats-monitor

# 4. Generate the Deployment manifest and configure it to use the SA
cat <<EOF > ~/stats-monitor/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: stats-monitor
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: stats-monitor
  template:
    metadata:
      labels:
        app: stats-monitor
    spec:
      serviceAccountName: stats-monitor-sa
      containers:
      - image: nginx:alpine
        name: nginx
EOF

# 5. Apply the initial Deployment
kubectl apply -f ~/stats-monitor/deployment.yaml

```

---

### **3. The Solution**

#### **Step 1: Disable Automounting on the Service Account**

You can either edit the Service Account directly or use a patch command.

**Option A (Edit):**

```bash
kubectl edit sa stats-monitor-sa -n monitoring

```

Add `automountServiceAccountToken: false` to the specification:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: stats-monitor-sa
  namespace: monitoring
automountServiceAccountToken: false

```

*Save and exit.*

**Option B (Patch - Faster for the exam):**

```bash
kubectl patch sa stats-monitor-sa -n monitoring -p '{"automountServiceAccountToken": false}'

```

#### **Step 2: Update the Deployment Manifest**

Now you need to manually project the token into the container using the provided manifest file.

1. **Open the manifest file:**
```bash
vi ~/stats-monitor/deployment.yaml

```


2. **Add the `volumeMounts` to the container:**
Locate the `containers` block and add the mount configuration, ensuring it is `readOnly: true`.
```yaml
      containers:
      - image: nginx:alpine
        name: nginx
        volumeMounts:
        - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          name: token
          readOnly: true

```


3. **Add the `volumes` to the pod spec:**
At the same indentation level as `containers`, add the `projected` volume block. The `path: token` setting ensures the file is created exactly as requested (`.../serviceaccount/token`).
```yaml
    spec:
      serviceAccountName: stats-monitor-sa
      volumes:
      - name: token
        projected:
          sources:
          - serviceAccountToken:
              path: token

```


*Save and exit.*

#### **Step 3: Apply and Verify**

1. **Apply the updated manifest:**
```bash
kubectl apply -f ~/stats-monitor/deployment.yaml

```


2. **Verify the configuration:**
Exec into the new pod to ensure the token exists exactly where required.
```bash
POD_NAME=$(kubectl get pods -n monitoring -l app=stats-monitor -o jsonpath='{.items[0].metadata.name}')

kubectl exec -it $POD_NAME -n monitoring -- ls -l /var/run/secrets/kubernetes.io/serviceaccount/token

```


*(You should see the `token` file listed).*

---

### **4. Official Documentation URLs (For the Exam)**

1. **Opt-out of API credential automounting:**
* **URL:** `https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#opt-out-of-api-credential-automounting`
* **Search Keyword:** Type `"automountServiceAccountToken"` in the search bar. This shows exactly where to place the field.


2. **ServiceAccount Token Volume Projection:**
* **URL:** `https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection`
* **Search Keyword:** Type `"projected"` or `"serviceAccountToken"` in the search bar. This provides the exact YAML structure for the `volumeMounts` and `volumes` blocks.


----

Here is an exam-style practical question based on your requirement, along with the step-by-step solution and the official Kubernetes documentation URLs.

### **The Question**

**Task:**

1. Create a new namespace named `secure-ns`.
2. Inside the `secure-ns` namespace, create a ServiceAccount named `app-sa`. Ensure that this ServiceAccount is explicitly configured so it **does not** automatically mount its API token into pods.
3. Create a Pod named `app-pod` in the `secure-ns` namespace using the `nginx:alpine` image.
4. The `app-pod` must use the `app-sa` ServiceAccount.
5. Manually project a ServiceAccount token for `app-sa` into the `app-pod`. The token must be mounted at the path `/var/run/secrets/tokens` with an expiration time of `3600` seconds (1 hour).

---

### **The Solution**

**Step 1: Create the Namespace and ServiceAccount**
First, create the namespace and generate the basic YAML for the ServiceAccount.

```bash
kubectl create ns secure-ns
kubectl create sa app-sa -n secure-ns -o yaml --dry-run=client > app-sa.yaml

```

Edit `app-sa.yaml` to disable automounting by adding `automountServiceAccountToken: false`:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-sa
  namespace: secure-ns
automountServiceAccountToken: false

```

Apply the file:

```bash
kubectl apply -f app-sa.yaml

```

**Step 2: Create the Pod with the Projected Volume**
Generate the basic Pod template:

```bash
kubectl run app-pod --image=nginx:alpine -n secure-ns -o yaml --dry-run=client > app-pod.yaml

```

Edit `app-pod.yaml` to specify the ServiceAccount and configure the projected token volume:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
  namespace: secure-ns
spec:
  serviceAccountName: app-sa
  containers:
  - image: nginx:alpine
    name: app-pod
    volumeMounts:
    - mountPath: /var/run/secrets/tokens
      name: custom-token-vol
  volumes:
  - name: custom-token-vol
    projected:
      sources:
      - serviceAccountToken:
          path: token
          expirationSeconds: 3600

```

Apply the file:

```bash
kubectl apply -f app-pod.yaml

```

**Step 3: Verify**
You can verify the token was correctly projected by checking the contents of the directory inside the running container:

```bash
kubectl exec -it app-pod -n secure-ns -- ls /var/run/secrets/tokens

```

*You should see a file named `token` inside the directory.*

---

### **Official Documentation URLs (For the Exam)**

During an exam like the CKA or CKS, you can search for the following concepts in the official docs to find exactly what you need to copy/paste:

1. **Opt-out of API credential automounting**
[https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#opt-out-of-api-credential-automounting](https://www.google.com/search?q=https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/%23opt-out-of-api-credential-automounting)
*(Search keyword: "automountServiceAccountToken: false")*
2. **ServiceAccount token volume projection**
[https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection](https://www.google.com/search?q=https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/%23serviceaccount-token-volume-projection)
*(Search keyword: "serviceAccountToken")*

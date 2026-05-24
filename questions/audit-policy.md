
Yes, to practice this specific scenario locally, you need a quick setup to mimic the "starting state" of the exam question.

Since the question states there is *already* a basic policy file located at `/etc/kubernetes/logpolicy/audit-policy.yaml` that "only specifies what not to log," you should create that file first before attempting the solution.

Here is the quick setup script to get your environment ready:

### **Lab Setup (Run this before trying the solution)**

You must run this on the **control plane (master) node** of a cluster provisioned with `kubeadm` (or a similar setup where you have root access to `/etc/kubernetes/`).

```bash
# 1. Create the directory for the policy
sudo mkdir -p /etc/kubernetes/logpolicy

# 2. Create the "basic" starting policy (Dropping the RequestReceived stage to reduce noise is a common "drop" rule)
sudo cat <<EOF > /etc/kubernetes/logpolicy/audit-policy.yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages:
  - "RequestReceived"
rules:
  # The basic policy only specifies what not to log
  - level: None
    users: ["system:kube-proxy"]
  - level: None
    users: ["system:apiserver"]
EOF

# 3. CRITICAL: Back up your API server manifest before starting the lab! 
# (Always do this in the real exam too)
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/kube-apiserver.yaml.bak

```

Once you have run that, your environment perfectly matches the prompt's starting conditions, and you can proceed with the solution I provided in the previous step!

### **Kubernetes Practice Lab: API Server Auditing**

This is a classic Certified Kubernetes Security Specialist (CKS) auditing scenario. Auditing requires meticulous attention to two areas: the `audit-policy.yaml` rules (where ordering matters) and the `kube-apiserver.yaml` manifest (where both flags and volume mounts must be precise).

Here is the step-by-step solution.

---

### **Step 1: Extend the Audit Policy File**

Audit policy rules are evaluated from top to bottom. The first rule that matches an event sets the audit level. Therefore, you must place your highly specific rules at the top and the generic "catch-all" rules at the bottom.

1. **Open the existing policy file:**
```bash
vi /etc/kubernetes/logpolicy/audit-policy.yaml

```


2. **Append the new rules below the existing "drop" rules:**
Keep whatever "drop" rules (`level: None`) are currently in the file at the very top. Add the requested rules immediately below them, in order of specificity.
```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
rules:
# [KEEP EXISTING 'level: None' RULES HERE]

# 1. Log the request body of deployments interactions in the namespace webapps
- level: RequestResponse
  resources:
  - group: "apps"
    resources: ["deployments"]
  namespaces: ["webapps"]

# 2. Log namespaces interactions at RequestResponse level
- level: RequestResponse
  resources:
  - group: ""
    resources: ["namespaces"]

# 3. Log ConfigMap and Secret interactions in all namespaces at the Metadata level
- level: Metadata
  resources:
  - group: ""
    resources: ["configmaps", "secrets"]

# 4. Log all other requests at the Metadata level (Catch-all MUST be at the bottom)
- level: Metadata

```


*Save and exit the file.*

---

### **Step 2: Reconfigure the `kube-apiserver` Manifest**

You must pass the configuration flags to the API server and mount the host directories into the static pod so the API server can read the policy and write the logs.

1. **Edit the static pod manifest:**
```bash
vi /etc/kubernetes/manifests/kube-apiserver.yaml

```


2. **Add the Audit Flags:**
Under the `spec.containers.command` section, add the following lines (ensure proper indentation):
```yaml
    - --audit-policy-file=/etc/kubernetes/logpolicy/audit-policy.yaml
    - --audit-log-path=/var/log/kubernetes/audit-logs.txt
    - --audit-log-maxage=10
    - --audit-log-maxbackup=2

```


3. **Add the Volume Mounts:**
Scroll down to `spec.containers.volumeMounts` and add mounts for both the policy directory and the log directory:
```yaml
    - mountPath: /etc/kubernetes/logpolicy
      name: audit-policy
      readOnly: true
    - mountPath: /var/log/kubernetes
      name: audit-logs
      readOnly: false

```


4. **Add the Volumes:**
Scroll to the very bottom to `spec.volumes` and add the host paths corresponding to the mounts:
```yaml
  - hostPath:
      path: /etc/kubernetes/logpolicy
      type: DirectoryOrCreate
    name: audit-policy
  - hostPath:
      path: /var/log/kubernetes
      type: DirectoryOrCreate
    name: audit-logs

```


*Save and exit the file.*

---

### **Step 3: Apply and Verify**

Because `kube-apiserver` is a static pod, saving the manifest file will automatically trigger Kubelet to restart it.

1. **Monitor the restart:**
```bash
watch crictl ps

```


*(Note: The prompt mentions using `docker ps` for troubleshooting if the cluster uses Docker Engine. If so, use `watch docker ps | grep kube-apiserver` to ensure the container comes back up).*
2. **Verify the logs are generating:**
Once the API server is back online, check the designated log path to confirm audit events are being written.
```bash
tail -f /var/log/kubernetes/audit-logs.txt

```



---

### **Official Documentation URLs (For the Exam)**

During the CKS exam, you can search for the following pages to copy standard templates:

1. **Audit Policy Rules Structure:**
* **URL:** `https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/#audit-policy`
* **Search Keyword:** Type `"Audit policy"` in the search bar. This section provides an extensive YAML example showing how to format rules for Secrets, ConfigMaps, and catch-all `Metadata` configurations.


2. **Audit Backend Flags (kube-apiserver):**
* **URL:** `https://kubernetes.io/docs/tasks/debug/debug-cluster/audit/#log-backend`
* **Search Keyword:** Type `"Log backend"` in the search bar. This provides the exact `--audit-log-*` flags needed for maxage, maxbackup, and pathing.

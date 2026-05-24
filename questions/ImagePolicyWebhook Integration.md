### **Kubernetes Practice Lab: ImagePolicyWebhook Integration**

This scenario tests your ability to configure the `ImagePolicyWebhook` admission controller, which is a critical topic on the CKS exam. It involves touching the API server, modifying an admission configuration file, and fixing a kubeconfig file that points to the webhook backend.

---

### **1. The Question (Exam Scenario)**

**Context:**
You must fully integrate a container image scanner into the kubeadm provisioned cluster.

**Task:**
Given an incomplete configuration located at `/etc/kubernetes/bouncer` and a functional container image scanner with an HTTPS endpoint at `https://smooth-yak.local/review`, perform the following tasks to implement a validating admission controller:

1. Reconfigure the API server to enable the `ImagePolicyWebhook` admission plugin and support the provided AdmissionConfiguration.
2. Reconfigure the ImagePolicyWebhook configuration to **deny** images on backend failure.
3. Complete the backend configuration to point to the container image scanner's endpoint at `https://smooth-yak.local/review`.
4. Finally, to test the configuration, deploy the test resource defined in `~/vulnerable.yaml` which is using an image that should be denied. (You may delete and re-create the resource as often as needed).

*(Note: The container image scanner's log file is located at `/var/log/nginx/access_log`)*

---

### **2. Lab Setup (Run this before trying the solution)**

Run this script on your **control plane node** to build the "broken/incomplete" starting state described in the prompt.

```bash
# 1. Create the configuration directories
sudo mkdir -p /etc/kubernetes/bouncer

# 2. Create the AdmissionConfiguration file (with defaultAllow incorrectly set to true)
sudo cat <<EOF > /etc/kubernetes/bouncer/admission_config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/bouncer/webhook-kubeconfig.yaml
      allowTTL: 50
      denyTTL: 50
      retryBackoff: 500
      defaultAllow: true
EOF

# 3. Create the Webhook Backend Kubeconfig (missing the server URL)
sudo cat <<EOF > /etc/kubernetes/bouncer/webhook-kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: ""  # Intentionally left blank for the lab
  name: image-scanner
contexts:
- context:
    cluster: image-scanner
    user: api-server
  name: image-scanner-context
current-context: image-scanner-context
preferences: {}
users:
- name: api-server
  user: {}
EOF

# 4. Create the vulnerable test pod
cat <<EOF > ~/vulnerable.yaml
apiVersion: v1
kind: Pod
metadata:
  name: vulnerable-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.14
EOF

# 5. Backup the API server manifest!
sudo cp /etc/kubernetes/manifests/kube-apiserver.yaml ~/kube-apiserver.yaml.bak

```

---

### **3. The Solution**

#### **Step 1: Update the AdmissionConfiguration (Deny on Failure)**

The prompt asks to deny images if the backend fails to respond. This maps to the `defaultAllow` parameter.

1. Edit the admission config file:
```bash
vi /etc/kubernetes/bouncer/admission_config.yaml

```


2. Change `defaultAllow: true` to `defaultAllow: false`:
```yaml
      defaultAllow: false

```


*Save and exit.*

#### **Step 2: Complete the Backend Webhook Kubeconfig**

The API server needs to know where to send the admission requests.

1. Edit the webhook kubeconfig file:
```bash
vi /etc/kubernetes/bouncer/webhook-kubeconfig.yaml

```


2. Add the URL provided in the prompt to the `server` field:
```yaml
clusters:
- cluster:
    insecure-skip-tls-verify: true
    server: "https://smooth-yak.local/review"

```


*Save and exit.*

#### **Step 3: Reconfigure the API Server**

Now, you must enable the plugin and point the API server to the AdmissionConfiguration.

1. Edit the static pod manifest:
```bash
vi /etc/kubernetes/manifests/kube-apiserver.yaml

```


2. Add the `ImagePolicyWebhook` to the existing `--enable-admission-plugins` flag. (If `NodeRestriction` is there, comma-separate them). Add the `--admission-control-config-file` flag.
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
    - --admission-control-config-file=/etc/kubernetes/bouncer/admission_config.yaml

```


3. Scroll down to `volumeMounts` and mount the `bouncer` directory so the API server can read both files:
```yaml
    - mountPath: /etc/kubernetes/bouncer
      name: bouncer-dir
      readOnly: true

```


4. Scroll down to `volumes` and create the host path:
```yaml
  - hostPath:
      path: /etc/kubernetes/bouncer
      type: DirectoryOrCreate
    name: bouncer-dir

```


*Save and exit.*

#### **Step 4: Verify and Test**

1. Wait for the API server to restart. You can monitor it using `watch crictl ps` or `watch docker ps | grep apiserver`.
2. Once the API server is up, apply the test resource:
```bash
kubectl apply -f ~/vulnerable.yaml

```


3. Because the mock URL (`https://smooth-yak.local/review`) does not actually exist in this lab environment, the backend will fail to respond. Since you set `defaultAllow: false`, the API server will reject the pod.
*You should see an error message similar to:*
`Error from server (Forbidden): error when creating "vulnerable.yaml": pods "vulnerable-pod" is forbidden: image policy webhook backend failed`

---

### **4. Official Documentation URLs (For the Exam)**

1. **Admission Configuration & Kubeconfig structure:**
* **URL:** `https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/#imagepolicywebhook`
* **Search Keyword:** Type `"ImagePolicyWebhook"` in the search bar. This page provides the exact layout for the `AdmissionConfiguration` block, the `kubeConfigFile` block, and the required API server flags.

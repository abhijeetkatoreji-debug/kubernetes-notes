### **Kubernetes Practice Lab: Securing the API Server**

This scenario is a classic CKS task that tests your ability to recover and secure a compromised or improperly configured API Server, and manage your `kubeconfig` contexts when authentication breaks.

---

### **1. The Question (Exam Scenario)**

**Context:**
For testing purposes, the kubeadm provisioned cluster’s API Server was configured to allow unauthenticated and unauthorized access.

**Task:**

1. Secure the cluster’s API server by configuring it as follows:
* Forbid anonymous authentication.
* Use authentication mode `Node,RBAC`. *(Note: The prompt says 'authentication mode' but technically means 'authorization mode')*
* Use admission controller `NodeRestriction`.


2. The cluster uses the Docker Engine as its container runtime. If needed, use the `docker` command to troubleshoot running containers. *(Use `crictl` if your environment uses containerd).*
3. `kubectl` is currently configured to use unauthenticated and unauthorized access. You do not have to change it, but be aware that `kubectl` will stop working once you have secured the cluster. You can use the cluster’s original configuration file located at `/etc/kubernetes/admin.conf` to access the secured cluster.
4. To clean up, remove the `ClusterRoleBinding` named `system:anonymous`.

---

### **2. Lab Setup (Run this before trying the solution)**

Run this script on your **control plane node** to deliberately make your cluster insecure, simulating the exact state described in the exam prompt.

```bash
# 1. Create the rogue ClusterRoleBinding that gives cluster-admin to anonymous users
kubectl create clusterrolebinding system:anonymous --clusterrole=cluster-admin --user=system:anonymous

# 2. Backup your good admin config so you don't lose it!
sudo cp /etc/kubernetes/admin.conf ~/admin.conf.bak
cp ~/.kube/config ~/.kube/config.bak

# 3. Make your default kubectl context anonymous (stripping out your client certs)
kubectl config set-credentials anonymous-user
kubectl config set-context anonymous-context --cluster=$(kubectl config view -o jsonpath='{.contexts[?(@.name=="'$(kubectl config current-context)'")].context.cluster}') --user=anonymous-user
kubectl config use-context anonymous-context

# 4. Insecure the API Server (Warning: This will restart the API server)
sudo sed -i 's/--authorization-mode=Node,RBAC/--authorization-mode=AlwaysAllow/g' /etc/kubernetes/manifests/kube-apiserver.yaml
sudo sed -i 's/--enable-admission-plugins=NodeRestriction/--enable-admission-plugins=AlwaysAdmit/g' /etc/kubernetes/manifests/kube-apiserver.yaml

```

*(Wait about 30 seconds for the API server to restart. If you run `kubectl get pods -A`, it should succeed, but you are now operating entirely as an anonymous, unauthenticated user!)*

---

### **3. The Solution**

#### **Step 1: Edit the API Server Manifest**

Because you are changing core API server parameters, you must edit the static pod manifest directly on the control plane node.

1. **Open the manifest file:**
```bash
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml

```


2. **Modify the command flags:**
Find the `spec.containers.command` block. Make the following changes:
* **Add:** `- --anonymous-auth=false`
* **Change:** `--authorization-mode=AlwaysAllow` to `- --authorization-mode=Node,RBAC`
* **Change:** `--enable-admission-plugins=AlwaysAdmit` to `- --enable-admission-plugins=NodeRestriction` (Keep any other plugins if they are comma-separated, just ensure `NodeRestriction` is there).


*Your flags should look something like this:*
```yaml
  containers:
  - command:
    - kube-apiserver
    - --advertise-address=192.168.1.10
    - --allow-privileged=true
    - --anonymous-auth=false
    - --authorization-mode=Node,RBAC
    - --enable-admission-plugins=NodeRestriction

```


*Save and exit (`:wq`).*

#### **Step 2: Wait for API Server Restart**

The Kubelet will detect the file change and automatically restart the API server container.

1. **Monitor the restart:**
```bash
watch crictl ps
# OR if strictly using Docker as per the prompt:
watch docker ps | grep kube-apiserver

```


2. Wait until you see a new `kube-apiserver` container running for at least 10-15 seconds.

#### **Step 3: Clean up using the Admin Kubeconfig**

If you try to run `kubectl get pods` now, it will fail with an `Unauthorized` error because you blocked anonymous access, and your default `kubectl` is still trying to be anonymous.

1. **Use the secure admin configuration to delete the rogue binding:**
Pass the `--kubeconfig` flag to authenticate as the real cluster administrator.
```bash
kubectl --kubeconfig /etc/kubernetes/admin.conf delete clusterrolebinding system:anonymous

```


2. **(Optional) Fix your local environment:**
To make your life easier for the rest of the exam, overwrite your broken default kubeconfig with the secure admin one so you don't have to keep typing the flag.
```bash
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown $(id -u):$(id -g) ~/.kube/config

```



---

### **4. Official Documentation URLs (For the Exam)**

During the exam, if you forget the exact syntax for the flags, use these references:

1. **Kube-apiserver CLI Arguments:**
* **URL:** `https://kubernetes.io/docs/reference/command-line-tools-reference/kube-apiserver/`
* **Search Keyword:** Type `kube-apiserver` in the search bar to find the CLI reference. You can `Ctrl+F` on that page for `anonymous-auth`, `authorization-mode`, or `enable-admission-plugins` to verify the exact spelling and syntax.


2. **Anonymous Requests:**
* **URL:** `https://kubernetes.io/docs/reference/access-authn-authz/authentication/#anonymous-requests`
* **Search Keyword:** Type `"Anonymous requests"` in the search bar to see how `--anonymous-auth=false` is implemented.

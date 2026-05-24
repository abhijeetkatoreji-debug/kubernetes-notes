Here is the complete lab setup script to prepare your cluster for this exact scenario.

I have also included a third namespace (`test-ns`) so you can verify that your NetworkPolicies actually block traffic from unauthorized namespaces, which is the best way to prove your solution works.

### **Lab Setup Script**

Run the following commands in your cluster to create the namespaces, labels, and deployments before you apply the NetworkPolicies.

```bash
# 1. Create the 'prod' namespace and label it
kubectl create ns prod
kubectl label ns prod env=prod

# 2. Create the 'data' namespace and label it
kubectl create ns data
kubectl label ns data env=data

# 3. Create a 'test-ns' namespace (to prove the policies block non-prod traffic)
kubectl create ns test-ns
kubectl label ns test-ns env=test

# 4. Create the target servers (and expose them to test ingress)
kubectl run prod-server --image=nginx:alpine -n prod --expose --port=80
kubectl run data-server --image=nginx:alpine -n data --expose --port=80

# 5. Create the client pods (to send traffic)
kubectl run prod-client --image=nginx:alpine -n prod -- sleep 3600
kubectl run test-client --image=nginx:alpine -n test-ns -- sleep 3600

# Wait a few seconds for the pods to be in the 'Running' state
kubectl get pods -A | grep -E 'prod|data|test-ns'

```

---

### **How to verify your work**

A great habit for the exam is to test the traffic flow **before** and **after** you apply the NetworkPolicies.

#### **Phase 1: Test BEFORE applying your solution**

At this stage, Kubernetes allows all traffic by default. Run these commands to verify that everything can communicate:

* **Test 1:** `prod-client` to `data-server` (Should **SUCCEED**)
```bash
kubectl exec prod-client -n prod -- curl -s -m 2 data-server.data.svc.cluster.local

```


* **Test 2:** `test-client` to `data-server` (Should **SUCCEED**)
```bash
kubectl exec test-client -n test-ns -- curl -s -m 2 data-server.data.svc.cluster.local

```


* **Test 3:** `test-client` to `prod-server` (Should **SUCCEED**)
```bash
kubectl exec test-client -n test-ns -- curl -s -m 2 prod-server.prod.svc.cluster.local

```



#### **Phase 2: Test AFTER applying your solution**

Once you apply the `deny-policy.yaml` and `allow-from-prod.yaml` from your solution, run the exact same tests. The results should now match the strict security requirements:

* **Test 1:** `prod-client` to `data-server` (Should still **SUCCEED** — allowed by `allow-from-prod`)
```bash
kubectl exec prod-client -n prod -- curl -s -m 2 data-server.data.svc.cluster.local

```


* **Test 2:** `test-client` to `data-server` (Should **FAIL/TIMEOUT** — blocked by `allow-from-prod`)
```bash
kubectl exec test-client -n test-ns -- curl -s -m 2 data-server.data.svc.cluster.local

```


* **Test 3:** `test-client` to `prod-server` (Should **FAIL/TIMEOUT** — blocked by `deny-policy` in prod)
```bash
kubectl exec test-client -n test-ns -- curl -s -m 2 prod-server.prod.svc.cluster.local

```



*(Note: The `-m 2` flag tells curl to timeout after 2 seconds, so you don't have to wait forever when a NetworkPolicy is successfully blocking your traffic).*

Would you like a setup script for any of the other security scenarios we've covered today?

Here is the exam-style scenario based on your requirements, complete with the step-by-step solution and the exact Kubernetes documentation URLs you can use during the exam.

### **The Question**

**Context:**
You must implement NetworkPolicies controlling the traffic flow of existing deployments across namespaces.

**Task:**

1. Create a NetworkPolicy named `deny-policy` in the `prod` namespace to block all ingress traffic.
*(Note: The `prod` namespace is labeled `env:prod`)*
2. Create a NetworkPolicy named `allow-from-prod` in the `data` namespace to allow ingress traffic only from Pods in the `prod` namespace. You must use the label of the `prod` namespace to allow this traffic.
*(Note: The `data` namespace is labeled `env:data`)*

**Constraints:**

* Do not modify or delete any existing namespaces or Pods.
* Only create the required NetworkPolicies.

---

### **The Solution**

#### **Step 1: Create the Default Deny Policy in the `prod` Namespace**

You need a "default deny all ingress" policy for the `prod` namespace. Since the policy needs to apply to all pods in the namespace, the `podSelector` is left empty.

1. **Create a file named `deny-policy.yaml`:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-policy
  namespace: prod
spec:
  podSelector: {}
  policyTypes:
  - Ingress

```


2. **Apply the policy:**
```bash
kubectl apply -f deny-policy.yaml

```



#### **Step 2: Create the Allow Policy in the `data` Namespace**

You need a policy in the `data` namespace that applies to all pods within it (`podSelector: {}`), but explicitly allows ingress traffic (`from`) based on the namespace label of the `prod` namespace (`namespaceSelector`).

1. **Create a file named `allow-from-prod.yaml`:**
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-prod
  namespace: data
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          env: prod

```


2. **Apply the policy:**
```bash
kubectl apply -f allow-from-prod.yaml

```



---

### **Official Documentation URLs (For the Exam)**

During the CKA or CKS exam, you should copy the base templates directly from the official docs to avoid syntax or indentation errors.

1. **For the "Deny All" Policy (Step 1):**
* **URL:** [https://kubernetes.io/docs/concepts/services-networking/network-policies/#default-deny-all-ingress-traffic](https://www.google.com/search?q=https://kubernetes.io/docs/concepts/services-networking/network-policies/%23default-deny-all-ingress-traffic)
* **Search Keyword:** Type `"Default deny all ingress traffic"` in the search bar. Copy the exact YAML provided in that section and just change the namespace to `prod`.


2. **For the "Allow by Namespace" Policy (Step 2):**
* **URL:** [https://kubernetes.io/docs/concepts/services-networking/network-policies/#networkpolicy](https://www.google.com/search?q=https://kubernetes.io/docs/concepts/services-networking/network-policies/%23networkpolicy)
* **Search Keyword:** Search for `"The NetworkPolicy resource"` or just `"namespaceSelector"`. You will find the massive, comprehensive NetworkPolicy example.
* **Strategy:** Copy the main structure, delete what you don't need (like `egress`, `ipBlock`, and `ports`), and modify the `namespaceSelector` block to match `env: prod`.

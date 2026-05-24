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

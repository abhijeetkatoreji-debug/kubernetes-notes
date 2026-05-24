Got it. Here is the practice lab and solution directly in the chat for you.

### **Kubernetes Practice Lab: Expose Web App with HTTPS Routes**

#### **1. The Question (Exam Scenario)**

**Context:**
You must expose a web application with HTTPS routes ensuring that all insecure traffic is automatically redirected to a secure connection.

**Task:**
Create an Ingress resource named `web` in the `prod` namespace and configure it as follows:

* Route traffic for host `web.k8s.local` and all paths to the existing Service `web`.
* Enable TLS termination using the existing Secret `web-cert`.
* Redirect HTTP requests to HTTPS.

**Verification:**
You can test your Ingress configuration with the following command:
`curl -kL http://web.k8s.local`

---

#### **2. Lab Setup (Run this before trying the solution)**

To practice this scenario, run the following commands in your cluster to create the required prerequisites (Namespace, Deployment, Service, and TLS Secret).

```bash
# 1. Create the namespace
kubectl create ns prod

# 2. Create the backend deployment and service
kubectl create deployment web --image=nginx:alpine -n prod
kubectl expose deployment web --port=80 --target-port=80 -n prod

# 3. Generate a self-signed certificate for the host web.k8s.local
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=web.k8s.local/O=web"

# 4. Create the TLS secret in the prod namespace
kubectl create secret tls web-cert --key tls.key --cert tls.crt -n prod

# 5. Clean up local files
rm tls.key tls.crt

# 6. Map the host to localhost in /etc/hosts (Assuming a local ingress controller like Minikube or kind)
# Note: You may need sudo for this step
echo "127.0.0.1 web.k8s.local" | sudo tee -a /etc/hosts

```

---

#### **3. The Solution**

**Step 1: Generate the Ingress YAML structure**
You can either copy the base template from the official Kubernetes documentation or use the declarative imperative command to get a starting point:

```bash
kubectl create ingress web -n prod \
  --rule="web.k8s.local/*=web:80,tls=web-cert" \
  -o yaml --dry-run=client > ingress.yaml

```

**Step 2: Edit the `ingress.yaml` file**
Open the file in your preferred editor (e.g., `vi ingress.yaml`). You will need to add the NGINX Ingress annotation to ensure HTTP to HTTPS redirection is enforced.

*(Note: NGINX Ingress controllers often redirect to HTTPS by default when a TLS section is provided, but explicitly defining the annotation ensures you meet the task requirements perfectly).*

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web
  namespace: prod
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - web.k8s.local
    secretName: web-cert
  rules:
  - host: web.k8s.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web
            port:
              number: 80

```

**Step 3: Apply the Ingress Resource**

```bash
kubectl apply -f ingress.yaml

```

**Step 4: Verify the Configuration**
Test the HTTP-to-HTTPS redirect as requested in the prompt. The `-L` flag tells `curl` to follow the redirect. The `-k` flag is added to allow self-signed certificates.

```bash
curl -kL -v http://web.k8s.local

```

*In the verbose (`-v`) output, you should see `HTTP/1.1 308 Permanent Redirect` followed by a successful connection to `https://web.k8s.local`.*

---

#### **4. Official Documentation URLs (For the Exam)**

During the exam, you can search the Kubernetes documentation for the base Ingress templates.

1. **Ingress TLS Configuration:**
* **URL:** `https://kubernetes.io/docs/concepts/services-networking/ingress/#tls`
* **Search Keyword:** Type `"Ingress TLS"` in the search bar. This section provides the exact YAML block for the `tls:` array (hosts and secretName).


2. **Ingress Annotations (Nginx):**
* **URL:** `https://kubernetes.github.io/ingress-nginx/user-guide/tls/`
* **Context:** Remember the annotation `nginx.ingress.kubernetes.io/ssl-redirect: "true"`. This enforces the redirection from HTTP to HTTPS.

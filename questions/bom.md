This is a classic Kubernetes security scenario, likely resembling a task from the CKS (Certified Kubernetes Security Specialist) exam.

Here is the step-by-step guide and the exact commands you need to fulfill all the requirements.

### **Step 1: Identify the Image with `libcrypto3` version `3.1.4-r5**`

First, you need to find out the names of the containers and their respective images running in the Deployment.

1. **List the containers and images:**
```bash
kubectl -n alpine get deploy alpine -o jsonpath='{range .spec.template.spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'

```


*This will output something like:*
```text
c1      alpine:3.17
c2      alpine:3.18
c3      alpine:3.19

```


2. **Check the package version in each container:**
Get the name of the running pod:
```bash
kubectl get pods -n alpine

```


Execute the `apk info` command inside each container (replace `<pod-name>` with your actual pod name, and `c1`, `c2`, `c3` with your actual container names):
```bash
kubectl exec <pod-name> -n alpine -c c1 -- apk info -v libcrypto3
kubectl exec <pod-name> -n alpine -c c2 -- apk info -v libcrypto3
kubectl exec <pod-name> -n alpine -c c3 -- apk info -v libcrypto3

```


**Identify the target:** Look at the output of those three commands. One of them will return exactly `libcrypto3-3.1.4-r5`. Note down the **image version** associated with that specific container. Let's assume it was `alpine:3.18` for the next steps.

---

### **Step 2: Generate the SPDX Document using `bom**`

Now that you have identified the vulnerable/target image (e.g., `alpine:3.18`), use the `bom` utility to generate the SPDX Software Bill of Materials (SBOM).

1. **Run the `bom generate` command:**
```bash
bom generate --image alpine:3.18 --format spdx --output ~/alpine.spdx

```


*(Note: Replace `alpine:3.18` with the actual image you identified in Step 1).*
2. **Verify the file was created successfully:**
```bash
ls -l ~/alpine.spdx
cat ~/alpine.spdx | head

```



---

### **Step 3: Update the Deployment Manifest**

Finally, you must remove the container running the identified image from the provided YAML manifest and apply the changes.

1. **Open the manifest file in a text editor:**
```bash
vi ~/alpine-deployment.yaml

```


2. **Delete the target container block:**
Navigate down to the `spec.template.spec.containers` section. Find the container block that uses your identified image (e.g., `image: alpine:3.18`).
Delete that **entire container block** (which usually includes `- name: ...`, `image: ...`, and possibly commands or ports). **Do not** touch the other two container blocks.
Save and exit (`:wq`).
3. **Apply the updated manifest:**
```bash
kubectl apply -f ~/alpine-deployment.yaml

```


4. **Verify the update:**
Check that the deployment has successfully rolled out and is now only running 2 containers instead of 3.
```bash
kubectl get pods -n alpine
kubectl describe deploy alpine -n alpine | grep Image:

```

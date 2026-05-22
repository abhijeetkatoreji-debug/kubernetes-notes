In Kubernetes, a **securityContext** defines the privilege and access control settings for a Pod or an individual Container. It acts as the boundary that dictates exactly what the containerized application is allowed to do on the host node's operating system.

While you can apply a security context at the Pod level, the **Container-level securityContext** contains specific fields to lock down individual processes.

Here is a breakdown of everything contained within a container's securityContext, grouped by what they control:

## 1. Identity and User Control

These fields determine *who* the process runs as, preventing applications from running as the default root user (UID 0).

* **runAsUser**: Specifies the User ID (UID) to run the entrypoint of the container process.
* **runAsGroup**: Specifies the Group ID (GID) to run the entrypoint of the container process.
* **runAsNonRoot**: A boolean safeguard (true/false). If set to true, the kubelet will check the image before starting the container. If the image tries to run as root (UID 0), the container will fail to start.

## 2. Privileges and Escalation

These settings control whether a container can gain elevated host-level permissions.

* **privileged**: A boolean. If true, the container is essentially given root access to the host machine. It can access all host devices (like /dev) and bypass almost all container isolation. *This should almost always be false.*
* **allowPrivilegeEscalation**: Controls whether a child process can gain more privileges than its parent process (typically via setuid or setgid binaries). Setting this to false is a best practice for most applications.

## 3. Linux Capabilities (capabilities)

Instead of granting full privileged access, Linux capabilities allow you to grant or remove specific granular root powers.

* **drop**: A list of capabilities to remove. A common security baseline is to drop ALL capabilities by default.
* **add**: A list of specific capabilities to grant back. For example, a container might need NET_BIND_SERVICE to bind to port 80, but doesn't need full root access to do so.

## 4. Filesystem Protections

* **readOnlyRootFilesystem**: If set to true, mounts the container's root filesystem as read-only. This means if an attacker compromises the application, they cannot download tools, modify scripts, or overwrite binaries. (The app must write any required temporary data to specific mounted emptyDir volumes instead).

## 5. Mandatory Access Control (MAC) and Filtering

These advanced fields hook into Linux kernel security modules to restrict the system calls a container can make.

* **seccompProfile**: Dictates which system calls (syscalls) the container is allowed to make to the Linux kernel. Usually set to RuntimeDefault to block dangerous system calls.
* **appArmorProfile**: Applies an AppArmor profile to restrict the capabilities of the container based on paths and process names.
* **seLinuxOptions**: Allows you to apply specific SELinux labels (User, Role, Type, Level) to the container to enforce strict isolation boundaries.

---

### What it looks like in practice

Here is an example of a highly secure, "least privilege" container securityContext:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-app
spec:
  containers:
  - name: my-app
    image: my-app:latest
    securityContext:
      runAsUser: 1000
      runAsGroup: 3000
      runAsNonRoot: true
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      privileged: false
      capabilities:
        drop:
          - ALL
      seccompProfile:
        type: RuntimeDefault

```

> **Pod vs. Container Contexts:** Note that a few settings, like fsGroup (which changes the ownership of mounted storage volumes) or sysctls (which modify kernel parameters), can *only* be set at the Pod-level securityContext, not the individual Container-level.

### **Kubernetes Practice Lab: Node and Docker Security**

This scenario tests your ability to secure the underlying container runtime on a Kubernetes node, a common topic in the CKS exam.

---

### **1. The Question (Exam Scenario)**

**Context:**
You have been asked to secure the underlying container runtime on a specific cluster node.

**Task:**
Perform the following tasks to secure the cluster node `cks-node-01`:

1. Remove the user `developer` from the `docker` group. (Do not remove any other user from any other group).
2. Reconfigure and restart the Docker daemon to ensure that the socket file located at `/var/run/docker.sock` is owned by the group `root`.
3. Reconfigure and restart the Docker daemon to ensure it does not listen on any TCP port.
4. After completing your work, ensure the Kubernetes cluster is healthy.

---

### **2. Lab Setup (Run this before trying the solution)**

Run this script on a node running Docker to simulate the insecure starting conditions described in the prompt. *(Note: This setup will briefly restart your Docker service).*

```bash
# 1. Create the dummy user and add them to the docker group
sudo useradd developer
sudo groupadd -f docker
sudo usermod -aG docker developer

# 2. Configure Docker insecurely (listening on TCP and using docker group)
sudo mkdir -p /etc/docker
sudo cat <<EOF | sudo tee /etc/docker/daemon.json
{
  "hosts": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"],
  "group": "docker"
}
EOF

# 3. Create a systemd drop-in so dockerd uses the daemon.json hosts
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo cat <<EOF | sudo tee /etc/systemd/system/docker.service.d/override.conf
[Service]
ExecStart=
ExecStart=/usr/bin/dockerd
EOF

# 4. Apply the insecure settings
sudo systemctl daemon-reload
sudo systemctl restart docker

```

---

### **3. The Solution**

#### **Step 1: Remove the user from the docker group**

You need to remove the `developer` user from the `docker` group without affecting their other group memberships.

1. **Verify the user's current groups:**
```bash
id developer

```


2. **Remove the user from the docker group:**
```bash
sudo gpasswd -d developer docker

```


*(Alternatively, `sudo deluser developer docker` works on Debian/Ubuntu-based systems).*
3. **Verify the removal:**
```bash
id developer

```


*(The `docker` group should no longer appear).*

#### **Step 2: Reconfigure the Docker Daemon**

Docker configuration is typically found in `/etc/docker/daemon.json` or within systemd service files (like `/lib/systemd/system/docker.service` or a drop-in file).

1. **Check the configuration file:**
```bash
sudo vi /etc/docker/daemon.json

```


2. **Modify the settings:**
* Remove the TCP entry from the `hosts` array.
* Change the `group` from `"docker"` to `"root"`.


**Before:**
```json
{
  "hosts": ["tcp://0.0.0.0:2375", "unix:///var/run/docker.sock"],
  "group": "docker"
}

```


**After:**
```json
{
  "hosts": ["unix:///var/run/docker.sock"],
  "group": "root"
}

```


*Save and exit.*

#### **Step 3: Restart Docker and Verify**

Apply the changes and ensure the daemon reflects your security updates.

1. **Restart the Docker service:**
```bash
sudo systemctl daemon-reload
sudo systemctl restart docker

```


2. **Verify TCP listening is disabled:**
```bash
sudo ss -tulnp | grep dockerd
# OR
sudo netstat -tulnp | grep dockerd

```


*(This should return no output, proving Docker is not listening on a TCP port).*
3. **Verify socket ownership:**
```bash
ls -l /var/run/docker.sock

```


*(The output should look similar to `srw-rw---- 1 root root ...`, confirming the group is now `root`).*
4. **Verify Cluster Health:**
Finally, check that the local Kubelet can still communicate with the runtime and the node is ready.
```bash
kubectl get nodes
kubectl get pods -A

```

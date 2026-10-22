# Kubelet Static Pods and CNI — Lab Notes

**Date:** 22 October 2026  
**VM:** Ubuntu 24.04 (`k8s-control-plane`), `192.168.57.10` (Vagrant/VirtualBox). Windows host can reach the VM.

## Objective

Run static Pods using **kubelet + containerd without Kubernetes Control Plane**, observe failure without CNI, configure a minimal CNI network, and verify a Pod's own IP.

## Environment

- kubelet `v1.34.12`, manually started for this lab.
- containerd `v2.2.1`, CRI socket `/run/containerd/containerd.sock`.
- `crictl v1.34.0`, `runc`, and CNI plugin binaries in `/opt/cni/bin/`.
- **No** API Server, etcd, Scheduler, Controller Manager, kubeadm, or kubectl.
- Swap was temporarily disabled (`sudo swapoff -a`).
- **Terminal 1:** foreground kubelet logs. **Terminal 2:** all other commands.

## 1. Run kubelet without Control Plane

The regular kubelet service was stopped:

```bash
sudo systemctl stop kubelet
```

Kubelet was started in **Terminal 1**:

```bash
sudo kubelet --pod-manifest-path=/etc/kubernetes/manifests
```

The flag is deprecated but works for this temporary lab. Kubelet watches the manifest directory and runs **static Pods** directly through the CRI runtime. No scheduler or API Server is required.

## 2. First Pod: Nginx on the host network

File: `/etc/kubernetes/manifests/nginx.yaml`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-static
  namespace: default
spec:
  hostNetwork: true
  containers:
    - name: nginx
      image: nginx:stable
      ports:
        - containerPort: 80
```

With `hostNetwork: true`, Nginx shares the VM's network and does not require CNI Pod networking.

Verified from the VM:

```bash
curl http://127.0.0.1:80
```

Verified from Windows:

```powershell
curl.exe http://192.168.57.10:80
```

Both returned the Nginx welcome page. The CRI sandbox and Nginx container were running.

## 3. Second Pod: HTTP Echo with its own network

File: `/etc/kubernetes/manifests/echo-network.yaml`

```bash
sudo tee /etc/kubernetes/manifests/echo-network.yaml > /dev/null <<'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: echo-network
  namespace: default
spec:
  containers:
    - name: echo
      image: hashicorp/http-echo:1.0
      args:
        - "-listen=:8080"
        - "-text=Hello from Pod network"
      ports:
        - containerPort: 8080
EOF
```

The file-writing command exits immediately. The existing kubelet process notices the file and attempts to start the Pod; no additional foreground process or terminal is needed.

**Observed kubelet error before CNI configuration:**

```text
Error syncing pod, skipping
network is not ready: container runtime network not ready:
NetworkReady=false reason:NetworkPluginNotReady
message:Network plugin returns error: cni plugin not initialized
```

Kubelet recognized the manifest, but the runtime could not prepare Pod networking, so no ready sandbox or running application container was created for `echo-network` at that point.

## 4. Diagnose CNI

```bash
ls -l /opt/cni/bin/
sudo ls -la /etc/cni/net.d/
```

**Findings:** `bridge`, `host-local`, `loopback`, `portmap`, and other CNI binaries were already installed. The configuration directory contained only an empty `.kubernetes-cni-keep` file. The missing component was **CNI configuration**, not the plugin binaries.

## 5. Configure minimal bridge networking

Created `/etc/cni/net.d/10-bridge.conflist`:

```bash
sudo tee /etc/cni/net.d/10-bridge.conflist > /dev/null <<'EOF'
{
  "cniVersion": "1.0.0",
  "name": "k8s-lab",
  "plugins": [
    {
      "type": "bridge",
      "bridge": "cni0",
      "isGateway": true,
      "ipMasq": true,
      "ipam": {
        "type": "host-local",
        "ranges": [[{"subnet": "10.244.0.0/24"}]],
        "routes": [{"dst": "0.0.0.0/0"}]
      }
    },
    {"type": "loopback"}
  ]
}
EOF
```

- **`bridge`** connects Pod networking to the Linux bridge `cni0`.
- **`host-local`** allocates IPs from `10.244.0.0/24`.
- **`isGateway`** makes the bridge a gateway for the Pod subnet.
- **`ipMasq`** requests masquerading for outbound traffic (not independently tested).
- **`loopback`** configures loopback inside the Pod network namespace.

This is a **single-node learning setup**, not a multi-node or production CNI solution.

## 6. Verify Pod sandboxes and containers

After writing the CNI configuration, kubelet successfully started the second Pod.

```bash
sudo crictl pods
```

Observed:

```text
POD ID         STATE   NAME                             NAMESPACE
fc648ca03eaea Ready   echo-network-k8s-control-plane   default
f76a489f06e02 Ready   nginx-static-k8s-control-plane   default
```

```bash
sudo crictl ps
```

Observed:

```text
CONTAINER       STATE     NAME    POD
43a77da44a3e9  Running   echo    echo-network-k8s-control-plane
6038a80408cee  Running   nginx   nginx-static-k8s-control-plane
```

**Concept:** `crictl pods` shows Pod sandboxes; `crictl ps` shows application containers. Containerd also manages the sandbox/pause container. `ctr -n k8s.io containers ls` and `ctr -n k8s.io tasks ls` expose lower-level containerd objects and tasks.

`crictl` warned about missing `/etc/crictl.yaml` and deprecated default endpoint probing. An explicit endpoint avoids probing:

```bash
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock pods
```

## 7. Inspect Pod IP and verify the application

```bash
sudo crictl inspectp fc648ca03eaea | grep -A 5 '"network"'
```

Relevant output:

```json
"network": {
  "additionalIps": [],
  "ip": "10.244.0.2"
}
```

From the Ubuntu VM:

```bash
curl http://10.244.0.2:8080
```

Response:

```text
Hello from Pod network
```

This confirms both the running application and network connectivity through its own Pod IP.

### Why a new `veth` appeared

With bridge-based CNI, a **veth pair** typically connects each Pod network namespace to the host bridge `cni0`: one endpoint is the Pod's `eth0`, and the other is on the host. Containers in the same Pod share the same network namespace and IP. Other CNI implementations can use different networking mechanisms.

## 8. Windows connectivity and Kubernetes Services

| Application | Address | From Ubuntu VM | From Windows |
|---|---|---|---|
| Nginx (`hostNetwork`) | `192.168.57.10:80` | Works | Works |
| HTTP Echo (Pod network) | `10.244.0.2:8080` | Works | Not reachable in this lab |

Windows has no configured route to `10.244.0.0/24` through the VM; forwarding/firewall settings may also need adjustment. **No additional bridge, Windows route, or forwarding configuration was added.**

A true Kubernetes `Service`/`NodePort` will be explored after setting up the necessary Kubernetes Control Plane and service networking. **CNI alone does not supply Kubernetes Services or DNS.**

## 9. Interview lessons

- `CNI plugin not initialized` means Pod networking could not be initialized; it does **not necessarily** mean the CNI binaries are missing.
- In OpenShift, check the configured network implementation (commonly OVN-Kubernetes), operator/component health, node status, CNI configuration, and container runtime logs.
- A Pod can remain **Pending** despite being assigned to a node when its sandbox cannot be prepared; inspect events and kubelet logs.
- `hostNetwork: true` bypasses ordinary Pod networking but is **not a general production workaround**: it changes isolation and can cause port conflicts.
- Distinguish **Pod object**, **Pod sandbox**, **application container**, and **network reachability**.

## 10. Current state and cleanup (NOT performed yet)

**Current state:**

- Terminal 1: kubelet running manually in foreground.
- Terminal 2: available for commands.
- Systemd kubelet service: stopped.
- Static Pods `nginx-static` and `echo-network`: both running.
- CNI configuration: `/etc/cni/net.d/10-bridge.conflist`.
- Pod subnet: `10.244.0.0/24`; HTTP Echo IP observed: `10.244.0.2`.

**When intentionally ending the lab:**

1. Remove **only the two experimental manifests** after verifying their contents: `/etc/kubernetes/manifests/nginx.yaml` and `/etc/kubernetes/manifests/echo-network.yaml`.
2. Allow kubelet to stop the Pods; confirm with `crictl pods` and `crictl ps`.
3. Press **Ctrl+C in Terminal 1** to stop manually running kubelet.
4. Decide whether to retain CNI for future labs. If restoring the original state, remove **only** `/etc/cni/net.d/10-bridge.conflist`; do not delete the CNI plugin binaries or the whole directory.
5. Restore the regular service: `sudo systemctl start kubelet`.
6. Check the desired swap policy separately; swap was disabled temporarily.

## 11. Key conclusions

1. Kubelet can run **static Pods without a Kubernetes Control Plane**.
2. A Pod with `hostNetwork: true` can run before ordinary Pod CNI is configured.
3. An ordinary Pod needs configured CNI networking for its own IP.
4. Installing CNI binaries and **configuring** CNI are different steps.
5. Pod sandbox readiness, application container state, and connectivity are separate checks.
6. Pod IP connectivity from the node does not automatically mean external Windows connectivity.
7. Kubernetes Services, DNS, and scheduler-driven workloads belong to later stages of this lab.

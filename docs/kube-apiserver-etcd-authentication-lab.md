# Kubernetes Control Plane Lab: etcd, kube-apiserver, Certificate Authentication, and Kubelet Registration

**Lab date:** 2026-10-22  
**Environment:** Ubuntu 24.04.3 LTS Vagrant VM (`k8s-control-plane`)  
**Kubernetes:** v1.34.12 (`kubelet`, `kube-apiserver`)  
**Container runtime:** containerd 2.2.1  
**Scope:** Manual, single-node, educational installation; **not** a production deployment.

> **Prerequisite:** This document continues the earlier `docs/kubelet-static-pods-cni-lab.md` exercise. The VM already has containerd, CNI plugins, a manually started kubelet, and two working static Pods (`nginx-static`, `echo-network`). The previous lab used `sudo kubelet --pod-manifest-path=/etc/kubernetes/manifests`.

## 1. Learning objectives

- Understand why the Kubernetes API Server needs etcd and ServiceAccount signing settings.
- Run `kube-apiserver` manually without installing kubeadm or creating a systemd unit.
- Distinguish server TLS certificates, client certificates, a Certificate Authority (CA), and ServiceAccount signing keys.
- Authenticate to the API Server using an X.509 client certificate.
- Build a kubelet-specific kubeconfig without `kubectl`.
- Register a Node and observe the creation of Mirror Pods for existing Static Pods.
- Understand which parts of a normal `kubeadm init` workflow are automated.

## 2. Architecture

### Before this lab

```text
Ubuntu Vagrant VM: k8s-control-plane

  kubelet --pod-manifest-path=/etc/kubernetes/manifests
      |
      v
  containerd 2.2.1
      |
      +-- nginx-static    (hostNetwork: true; port 80)
      +-- echo-network    (CNI bridge; Pod IP 10.244.0.2; port 8080)

No kube-apiserver connection, no registered Node, no Mirror Pods.
```

### After this lab

```text
                         Admin client (curl)
                       admin.crt + admin.key
                                |
                                | HTTPS / TLS client authentication
                                v
                  +-----------------------------+
                  | kube-apiserver :6443        |
                  | --client-ca-file=...        |
                  | API authentication          |
                  +--------------+--------------+
                                 |
                                 | HTTP localhost:2379 (LAB ONLY)
                                 v
                            +---------+
                            |  etcd   |
                            +---------+
                                 ^
                                 | Node and Mirror Pod API objects
                                 |
  +------------------------------+-------------------------------+
  |                                                              |
  | kubelet --kubeconfig=/etc/kubernetes/kubelet.conf              |
  |   identity: CN=system:node:k8s-control-plane                   |
  |             O=system:nodes                                    |
  |   |                                                          |
  |   +--> containerd --> nginx-static                            |
  |                    --> echo-network                           |
  |                                                              |
  | Node: k8s-control-plane (Ready=True)                          |
  | Mirror Pods visible in API: both Running                      |
  +--------------------------------------------------------------+

No scheduler, controller-manager, kube-proxy, CoreDNS, or kubeadm.
```

**Key distinction:** kubelet and containerd run the containers. API Server exposes and persists cluster state in etcd. Static Pods are managed from local manifests, while *Mirror Pods* are their representations in the API.

## 3. Relevant lab addresses and paths

| Item | Value |
|---|---|
| VM hostname | `k8s-control-plane` |
| VM NAT IP reported by Node | `10.0.2.15` |
| VM host-only IP | `192.168.57.10` |
| API Server | `https://127.0.0.1:6443` |
| etcd (local only) | `http://127.0.0.1:2379` |
| Static Pod manifests | `/etc/kubernetes/manifests` |
| kubelet kubeconfig | `/etc/kubernetes/kubelet.conf` |
| API Server log | `/tmp/kube-apiserver.log` |
| Temporary certificates and keys | `/var/run/kubernetes/` |

**Persistence warning:** `/var/run` is volatile. These lab credentials and self-generated API Server TLS files may disappear after a reboot. Do not rely on this layout for a persistent cluster.

## 4. Install the kube-apiserver binary

The `kube-apiserver` executable was downloaded directly; `apt-get install kube-apiserver` was attempted but the package was not available from the configured repositories.

```bash
curl -fL https://dl.k8s.io/v1.34.12/bin/linux/amd64/kube-apiserver -o /tmp/kube-apiserver
sudo install -m 755 /tmp/kube-apiserver /usr/local/bin/kube-apiserver
```

Initial execution without `sudo` failed to create `/var/run/kubernetes` (`permission denied`). Running it with `sudo` produced self-signed API Server TLS files at `/var/run/kubernetes/apiserver.crt` and `/var/run/kubernetes/apiserver.key`, then exposed required startup arguments:

```text
--etcd-servers must be specified
service-account-issuer is a required flag
either --service-account-key-file or --service-account-signing-endpoint must be set
--service-account-signing-key-file and --service-account-issuer are required flags
```

Before the service was running, both checks demonstrated the absence of an API listener:

```bash
curl -k -v https://127.0.0.1:6443/livez
ss -lntp | grep :6443
pgrep -a kube-apiserver
```

**Observed:** `Connection refused`, no listener, and no API Server process.

## 5. Install and verify etcd

```bash
sudo apt-get update && sudo apt-get install -y etcd-server
```

The package started etcd as a systemd service. It listened on localhost port `2379` using **unencrypted HTTP** in this lab.

```bash
curl -v http://127.0.0.1:2379/
curl http://127.0.0.1:2379/health
```

**Observed:** the root path returned HTTP `404` (not a health failure); `/health` returned:

```json
{"health":"true"}
```

With only the etcd flag supplied:

```bash
sudo kube-apiserver --etcd-servers=http://127.0.0.1:2379
```

**Observed:** the etcd-servers validation error disappeared, but the ServiceAccount signing/issuer requirements remained.

## 6. Generate ServiceAccount signing keys

These keys are **not** the HTTPS server certificate and are **not** client authentication certificates. They sign and verify Kubernetes ServiceAccount tokens.

```bash
sudo openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out /var/run/kubernetes/sa.key
sudo openssl pkey -in /var/run/kubernetes/sa.key -pubout -out /var/run/kubernetes/sa.pub
```

The chosen issuer is an **issuer identifier**, not necessarily a browsable URL:

```text
https://kubernetes.default.svc
```

`kubernetes.default.svc` is also a standard in-cluster Service DNS name. This lab does not yet have CoreDNS or the full Service infrastructure; opening it in Chrome on the Windows host is not expected to work.

## 7. Start kube-apiserver in the background

We deliberately avoided creating a new systemd service for this temporary exercise. The final launch command (after configuring client certificate trust in section 9) was:

```bash
sudo nohup /usr/local/bin/kube-apiserver \
  --etcd-servers=http://127.0.0.1:2379 \
  --service-account-issuer=https://kubernetes.default.svc \
  --service-account-signing-key-file=/var/run/kubernetes/sa.key \
  --service-account-key-file=/var/run/kubernetes/sa.pub \
  --client-ca-file=/var/run/kubernetes/client-ca.crt \
  > /tmp/kube-apiserver.log 2>&1 < /dev/null &
```

**Note on chronology:** initially, the same background command was used **without** `--client-ca-file`; after creating the CA and admin client certificate, the API Server was stopped and restarted with that flag.

```bash
sudo pkill -x kube-apiserver
```

The stop command was used before restarting with the updated arguments, **not** after the final successful launch.

A background process is temporary and is not managed or automatically restarted by systemd. The API Server log is at `/tmp/kube-apiserver.log`.

## 8. First API responses: 401 Unauthorized

```bash
curl -k https://127.0.0.1:6443/livez
curl -k -i https://127.0.0.1:6443/version
```

**Observed:** HTTP `401 Unauthorized` without client credentials. This proved the API Server was reachable and processing HTTPS requests, but did not establish an authenticated user.

- **401 Authentication:** no accepted identity/credentials.
- **403 Authorization:** an identity is known but is not permitted to perform an operation.

`curl -k` disables verification of the **server** TLS certificate. It does not authenticate the client.

## 9. Create a client CA and administrator certificate

### 9.1 Client CA

```bash
sudo openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out /var/run/kubernetes/client-ca.key
sudo openssl req -x509 -new -key /var/run/kubernetes/client-ca.key -sha256 -days 365 -subj "/CN=Kubernetes Lab Client CA" -out /var/run/kubernetes/client-ca.crt
```

The API Server trusts certificates signed by this CA when launched with:

```text
--client-ca-file=/var/run/kubernetes/client-ca.crt
```

### 9.2 Administrator key and CSR

```bash
sudo openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out /var/run/kubernetes/admin.key
sudo openssl req -new -key /var/run/kubernetes/admin.key -out /var/run/kubernetes/admin.csr -subj "/CN=kubernetes-admin/O=system:masters"
```

- `CN=kubernetes-admin`: username extracted from the certificate.
- `O=system:masters`: highly privileged Kubernetes group. **Lab only.**

### 9.3 Client authentication extension and certificate signing

The first attempt passed a Bash process substitution to `sudo openssl`:

```bash
# FAILED ATTEMPT — retained only as troubleshooting history
sudo openssl x509 ... -extfile <(printf 'extendedKeyUsage=clientAuth')
```

**Observed:** OpenSSL could not open `/dev/fd/63` because that file descriptor was not available across the `sudo` boundary. The fix was a real temporary extension file:

```bash
printf 'extendedKeyUsage=clientAuth\n' | sudo tee /tmp/admin-cert.ext > /dev/null
```

Sign the admin CSR:

```bash
sudo openssl x509 -req \
  -in /var/run/kubernetes/admin.csr \
  -CA /var/run/kubernetes/client-ca.crt \
  -CAkey /var/run/kubernetes/client-ca.key \
  -CAcreateserial \
  -out /var/run/kubernetes/admin.crt \
  -days 365 -sha256 \
  -extfile /tmp/admin-cert.ext
```

**Observed:**

```text
Certificate request self-signature ok
subject=CN = kubernetes-admin, O = system:masters
```

Verify the CA chain:

```bash
sudo openssl verify -CAfile /var/run/kubernetes/client-ca.crt /var/run/kubernetes/admin.crt
```

**Observed:** `/var/run/kubernetes/admin.crt: OK`.

The API Server was then restarted with the `--client-ca-file` argument (see section 7).

## 10. Authenticate to the Kubernetes API with curl

```bash
sudo curl -k -i \
  --cert /var/run/kubernetes/admin.crt \
  --key /var/run/kubernetes/admin.key \
  https://127.0.0.1:6443/version
```

**Observed:** `HTTP/2 200` and JSON reporting `gitVersion: v1.34.12`, `platform: linux/amd64`.

```bash
sudo curl -k -i \
  --cert /var/run/kubernetes/admin.crt \
  --key /var/run/kubernetes/admin.key \
  https://127.0.0.1:6443/api/v1/namespaces
```

**Observed:** standard namespaces existed: `default`, `kube-system`, `kube-public`, `kube-node-lease`.

At this stage, an API request for Pods showed an empty list. The kubelet had not yet been configured to connect to the API Server.

```bash
sudo curl -ks \
  --cert /var/run/kubernetes/admin.crt \
  --key /var/run/kubernetes/admin.key \
  https://127.0.0.1:6443/api/v1/nodes
```

**Observed before kubelet registration:** `"items": []`.

**Authorization caveat:** this manual API Server was not configured with an explicit RBAC authorization mode. Successful requests do **not** demonstrate that RBAC granted access. Certificate authentication and authorization are different stages.

## 11. Generate a kubelet client identity

The kubelet must not reuse the administrator certificate. It gets a node-specific identity:

```text
CN=system:node:k8s-control-plane
O=system:nodes
```

```bash
sudo openssl genpkey -algorithm RSA -pkeyopt rsa_keygen_bits:2048 -out /var/run/kubernetes/kubelet.key
sudo openssl req -new \
  -key /var/run/kubernetes/kubelet.key \
  -out /var/run/kubernetes/kubelet.csr \
  -subj "/CN=system:node:k8s-control-plane/O=system:nodes"
```

The same client-authentication extension file is appropriate:

```bash
sudo openssl x509 -req \
  -in /var/run/kubernetes/kubelet.csr \
  -CA /var/run/kubernetes/client-ca.crt \
  -CAkey /var/run/kubernetes/client-ca.key \
  -CAcreateserial \
  -out /var/run/kubernetes/kubelet.crt \
  -days 365 -sha256 \
  -extfile /tmp/admin-cert.ext
```

**Observed:**

```text
Certificate request self-signature ok
subject=CN = system:node:k8s-control-plane, O = system:nodes
```

## 12. Create kubelet.conf manually (kubectl was not installed)

An attempted `kubectl config ...` workflow could not be used because `kubectl` was not installed in the VM. No installation was necessary: a kubeconfig is YAML.

```bash
sudo tee /etc/kubernetes/kubelet.conf > /dev/null <<'EOF_KUBECONFIG'
apiVersion: v1
kind: Config
clusters:
- name: kubernetes
  cluster:
    server: https://127.0.0.1:6443
    insecure-skip-tls-verify: true
users:
- name: kubelet
  user:
    client-certificate: /var/run/kubernetes/kubelet.crt
    client-key: /var/run/kubernetes/kubelet.key
contexts:
- name: kubelet-context
  context:
    cluster: kubernetes
    user: kubelet
current-context: kubelet-context
EOF_KUBECONFIG
```

`insecure-skip-tls-verify: true` was a deliberate **temporary lab shortcut** because the API Server had generated a self-signed TLS certificate. Production clients must verify the API Server's certificate against a trusted CA and use an appropriate hostname/IP in the certificate SAN.

## 13. Restart kubelet with API Server access

The original kubelet had been launched in a terminal with:

```bash
sudo kubelet --pod-manifest-path=/etc/kubernetes/manifests
```

The replacement command was:

```bash
sudo kubelet \
  --pod-manifest-path=/etc/kubernetes/manifests \
  --kubeconfig=/etc/kubernetes/kubelet.conf
```

### Troubleshooting: port 10250 already in use

The first replacement attempt failed:

```text
Failed to listen and serve
listen tcp 0.0.0.0:10250: bind: address already in use
```

We inspected the owner of the port:

```bash
sudo ss -lntp '( sport = :10250 )'
```

**Observed:** an earlier kubelet process, PID `3946`, still listened on port `10250`.

```bash
sudo kill 3946
```

The user then verified the port was free and relaunched kubelet with `--kubeconfig`. The successful process had PID `6145`. A second port conflict occurred during the transition; the user cleared it before the successful launch. **Do not assume `Ctrl+C` stopped the original process: verify the actual listener when a bind error occurs.**

### Successful kubelet log evidence

```text
Attempting to register node    node="k8s-control-plane"
Successfully registered node   node="k8s-control-plane"
Creating a mirror pod for static pod   pod="default/echo-network-k8s-control-plane"
Creating a mirror pod for static pod   pod="default/nginx-static-k8s-control-plane"
Fast updating node status as it just became ready
Watching apiserver
```

Early messages such as `node "k8s-control-plane" not found`, `PLEG is not healthy: pleg has yet to be successful`, and the absence of the CRI-O socket occurred during startup and did **not** prevent successful registration. containerd was the selected and initialized runtime.

## 14. Verify the registered Node

```bash
sudo curl -ks \
  --cert /var/run/kubernetes/admin.crt \
  --key /var/run/kubernetes/admin.key \
  https://127.0.0.1:6443/api/v1/nodes
```

**Observed:** one Node named `k8s-control-plane`, with:

```text
Ready:                  True
MemoryPressure:         False
DiskPressure:           False
PIDPressure:            False
CPU capacity:           2
Memory capacity:        4007908Ki (~3.82 GiB)
Container runtime:      containerd://2.2.1
Kubelet version:        v1.34.12
InternalIP:             10.0.2.15
Kubelet HTTPS endpoint: 10250
```

**Interesting observation:** `spec.taints` still contained `node.kubernetes.io/not-ready:NoSchedule` even though `Ready=True`. The node controller normally reconciles such taints, but the controller-manager was not running in this lab. The readiness condition and taint are separate fields.

## 15. Verify Mirror Pods

```bash
sudo curl -ks \
  --cert /var/run/kubernetes/admin.crt \
  --key /var/run/kubernetes/admin.key \
  https://127.0.0.1:6443/api/v1/namespaces/default/pods \
  | python3 -c 'import json,sys; d=json.load(sys.stdin); [print(p["metadata"]["name"], p["status"]["phase"]) for p in d["items"]'
```

**Observed:**

```text
echo-network-k8s-control-plane Running
nginx-static-k8s-control-plane Running
```

This verifies the path from kubelet to API Server and etcd. The running containers themselves remain controlled by kubelet and containerd.

## 16. Authentication and identity: what each file does

| File | Role |
|---|---|
| `/var/run/kubernetes/apiserver.crt` | API Server HTTPS server certificate (auto-generated in lab) |
| `/var/run/kubernetes/apiserver.key` | API Server HTTPS private key |
| `/var/run/kubernetes/sa.key` | Private key for signing ServiceAccount tokens |
| `/var/run/kubernetes/sa.pub` | Public key for validating ServiceAccount tokens |
| `/var/run/kubernetes/client-ca.key` | Private CA key used to sign client certificates; protect carefully |
| `/var/run/kubernetes/client-ca.crt` | Trusted client CA certificate supplied to API Server |
| `/var/run/kubernetes/admin.key` | Admin client private key |
| `/var/run/kubernetes/admin.crt` | Admin client certificate (`kubernetes-admin`, `system:masters`) |
| `/var/run/kubernetes/kubelet.key` | Kubelet client private key |
| `/var/run/kubernetes/kubelet.crt` | Kubelet client certificate (`system:node:k8s-control-plane`, `system:nodes`) |
| `/etc/kubernetes/kubelet.conf` | API endpoint and kubelet client credentials |

**Certificate flow:**

```text
client-ca.key + client-ca.crt
       |
       +-- signs admin.csr   --> admin.crt   --> curl authenticates
       |
       +-- signs kubelet.csr --> kubelet.crt --> kubelet authenticates

kube-apiserver --client-ca-file=client-ca.crt
       |
       +-- verifies certificate chain and extracts CN/O identity
```

**ServiceAccount tokens are separate:** the issuer and `sa.key`/`sa.pub` relate to signed tokens used by workloads, not the X.509 client certificates used in these exercises.

## 17. What kubeadm would normally automate

In a typical `kubeadm init` deployment, kubeadm prepares certificates, kubeconfig files, control-plane static Pod manifests, and initial cluster configuration. Kubelet bootstrapping and certificate rotation are also part of the supported kubeadm-based workflow. Here we intentionally assembled the components by hand to understand each dependency.

This exercise is **not** a substitute for kubeadm or a recommended production installation pattern. Before switching to kubeadm on the same VM, plan a controlled cleanup of manual processes, temporary credentials, the manual kubelet configuration, and any conflicting etcd/API Server state. `kubeadm reset` alone should not be assumed to remove every manually created artifact.

## 18. Limitations and security warnings

- etcd uses **unencrypted HTTP** on `127.0.0.1:2379`; do not expose this endpoint.
- API Server uses an auto-generated self-signed server certificate; client commands use `curl -k` and kubelet uses `insecure-skip-tls-verify: true`.
- API Server was not explicitly configured for RBAC/Node authorization; **do not interpret this as a secure, production-ready authorization setup**. In particular, `system:nodes` identity alone does not enforce Node authorizer restrictions.
- `system:masters` is highly privileged and should not be used casually.
- Keys under `/var/run/kubernetes` are volatile; credentials and certificates will not survive normal reboot semantics.
- No controller-manager, scheduler, kube-proxy, CoreDNS, full Service networking, or certificate rotation was configured.
- `nohup` is a temporary process-management convenience, not production service management.
- Never commit private keys, kubeconfig files containing credentials, or certificates containing sensitive information to GitHub. **This document contains commands and paths only, not key material.**

## 19. Final result and next module

**Successfully demonstrated:**

```text
etcd health                      healthy
kube-apiserver                   responding on HTTPS :6443
Unauthenticated /version         401
Authenticated /version           HTTP/2 200 (v1.34.12)
Standard namespaces              visible
Node registration                k8s-control-plane
Node Ready                       True
Mirror Pods                      2 Running
Existing static workloads        preserved
```

**Next logical module:** `kube-controller-manager` and controller reconciliation, including the difference between Node `Ready` and the lingering `node.kubernetes.io/not-ready` taint. Keep it a separate, focused lab.

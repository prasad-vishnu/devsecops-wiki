# Kubernetes: Installation and Setup

| | |
|---|---|
| **Cluster** | 04 Containers & IDP |
| **Role** | Anchor |
| **Prerequisites** | Docker Engine or Docker Desktop running locally (8 GB RAM free recommended), a POSIX shell (Git Bash or WSL2 on Windows), AWS CLI v2 configured with SSO for the EKS section |
| **Target version** | Kubernetes 1.31+ (kind node image `kindest/node:v1.31.x`), kubectl within one minor version of the cluster |
| **Capstone phase** | Phase 3 (Months 9-12) |

> After this file you will have a repeatable, scripted local Kubernetes environment: a three-node kind cluster running Cilium (so NetworkPolicy is enforced), metrics-server (so HPA and `kubectl top` work), the Gateway API CRDs with Envoy Gateway, and a local OCI registry wired into the cluster. You will also know how to authenticate to Amazon EKS with short-lived IAM credentials, how kubeconfig contexts work, and how to verify every component before starting the labs. Everything here is captured in a `Makefile` so the cluster can be destroyed and rebuilt in five minutes.

---

## 1. What You Are Installing and Why

| Tool | Purpose | Required for labs |
|---|---|---|
| `kubectl` | The CLI client to the Kubernetes API. One binary, talks to any cluster | Yes |
| `kind` | "Kubernetes in Docker". Runs each Kubernetes node as a Docker container. Multi-node, upstream-conformant, supports custom CNI. Our primary local cluster | Yes |
| `k3d` | Runs k3s (a lightweight CNCF-certified distribution) in Docker. Faster startup, lower memory, bundled Traefik and local-path storage. Alternative to kind for low-RAM laptops | Optional |
| `helm` | Package manager. Needed to install Cilium, metrics-server, Envoy Gateway, and other add-ons even before you study Helm itself | Yes |
| `kubectx` / `kubens` | Switch kubeconfig contexts and namespaces quickly | Recommended |
| `k9s` | Terminal UI for live cluster inspection | Recommended |
| `stern` | Multi-Pod log tailing with colour and regex | Recommended |
| `cilium` CLI | Install and validate Cilium; `cilium connectivity test` | Yes (for the NetworkPolicy labs) |
| `aws` CLI v2 + `eksctl` | EKS authentication and (optionally) quick EKS clusters outside Terraform | For the cloud section |
| `kustomize` | Bundled into `kubectl kustomize` / `kubectl apply -k`; the standalone binary is newer | Optional |

Pin `kubectl` to within one minor version of your cluster. The version skew policy allows `kubectl` to be one minor version older or newer than the API server (for example kubectl 1.30, 1.31, or 1.32 with a 1.31 cluster).

---

## 2. Install the Tooling

### 2.1 Linux (Debian/Ubuntu, amd64)

```bash
# kubectl (official apt repo, pinned to the 1.31 minor)
sudo apt-get update && sudo apt-get install -y apt-transport-https ca-certificates curl gnupg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key \
  | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' \
  | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update && sudo apt-get install -y kubectl

# kind (single static binary)
KIND_VERSION=v0.24.0
curl -Lo ./kind "https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-amd64"
chmod +x ./kind && sudo mv ./kind /usr/local/bin/kind

# k3d (optional alternative)
curl -s https://raw.githubusercontent.com/k3d-io/k3d/main/install.sh | bash

# helm
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# kubectx + kubens
sudo git clone https://github.com/ahmetb/kubectx /opt/kubectx
sudo ln -sf /opt/kubectx/kubectx /usr/local/bin/kubectx
sudo ln -sf /opt/kubectx/kubens  /usr/local/bin/kubens

# cilium CLI
CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/main/stable.txt)
curl -L --fail --remote-name-all \
  "https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-amd64.tar.gz"
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin && rm cilium-linux-amd64.tar.gz

# k9s and stern via their GitHub releases, or:
# sudo snap install k9s
# brew install stern   (Linuxbrew)
```

### 2.2 macOS (Homebrew, Apple Silicon or Intel)

```bash
brew install kubectl kind k3d helm kubectx k9s stern cilium-cli kustomize awscli eksctl
```

Docker Desktop on macOS must be allocated at least 6 GB memory and 4 CPUs for the three-node kind cluster (Settings, Resources). Colima or OrbStack also work as the Docker host.

### 2.3 Windows 11

Two supported approaches. **Recommended: WSL2** (Ubuntu) with Docker Desktop's WSL2 backend; then follow the Linux steps inside WSL2. Everything in this wiki assumes a POSIX shell.

If you must stay in native Windows, use `winget` from PowerShell:

```powershell
winget install -e --id Kubernetes.kubectl
winget install -e --id Kubernetes.kind
winget install -e --id Helm.Helm
winget install -e --id ahmetb.kubectx
winget install -e --id ahmetb.kubens
winget install -e --id Derailed.k9s
winget install -e --id stern.stern
winget install -e --id Amazon.AWSCLI
```

Then run the shell snippets below in Git Bash. Note that `kind` on native Windows cannot run Cilium without the WSL2 backend, because Cilium needs a modern Linux kernel with eBPF support under the Docker VM; Docker Desktop with WSL2 provides this.

### 2.4 Containers (CI runners and devcontainers)

A Dockerfile fragment for a devcontainer or GitHub Actions self-hosted runner image:

```dockerfile
FROM mcr.microsoft.com/devcontainers/base:ubuntu-24.04
ARG KUBECTL_VERSION=v1.31.4
ARG KIND_VERSION=v0.24.0
ARG HELM_VERSION=v3.16.3
RUN curl -fsSLo /usr/local/bin/kubectl "https://dl.k8s.io/release/${KUBECTL_VERSION}/bin/linux/amd64/kubectl" \
 && curl -fsSLo /usr/local/bin/kind "https://kind.sigs.k8s.io/dl/${KIND_VERSION}/kind-linux-amd64" \
 && curl -fsSL "https://get.helm.sh/helm-${HELM_VERSION}-linux-amd64.tar.gz" | tar xz -C /tmp \
 && mv /tmp/linux-amd64/helm /usr/local/bin/helm \
 && chmod +x /usr/local/bin/kubectl /usr/local/bin/kind /usr/local/bin/helm
USER vscode
```

In GitHub Actions, the maintained actions `azure/setup-kubectl@v4`, `helm/kind-action@v1`, and `azure/setup-helm@v4` do the same thing; the kind action spins up an ephemeral cluster for integration tests, which the capstone CI uses in Phase 5.

### 2.5 Verify the binaries

```bash
kubectl version --client --output=yaml
kind version
helm version --short
cilium version --client
docker info --format '{{.ServerVersion}} {{.OSType}}/{{.Architecture}} CPUs={{.NCPU}} Mem={{.MemTotal}}'
```

Expected output (versions will differ):

```text
clientVersion:
  gitVersion: v1.31.4
  platform: linux/amd64
kustomizeVersion: v5.4.2
kind v0.24.0 go1.22.6 linux/amd64
v3.16.3+gcfd0749
cilium-cli: v0.16.20 compiled with go1.23.2 on linux/amd64
27.3.1 linux/x86_64 CPUs=8 Mem=16777216000
```

---

## 3. Shell Setup

Add to `~/.bashrc` or `~/.zshrc`:

```bash
# Completion
source <(kubectl completion bash)     # or zsh
source <(helm completion bash)
source <(kind completion bash)

# The alias everyone uses, with completion preserved
alias k=kubectl
complete -o default -F __start_kubectl k

# Keep kubeconfigs per cluster and merge them; avoids one giant file
export KUBECONFIG="$HOME/.kube/config:$HOME/.kube/kind-capstone.yaml:$HOME/.kube/eks-capstone.yaml"

# Never accidentally run against production: show context in the prompt
# (kube-ps1 via brew/apt provides the kube_ps1 function)
# PS1='$(kube_ps1) '$PS1
```

Install the `krew` plugin manager and a few plugins that the troubleshooting file relies on:

```bash
(
  set -x; cd "$(mktemp -d)" &&
  OS="$(uname | tr '[:upper:]' '[:lower:]')" &&
  ARCH="$(uname -m | sed -e 's/x86_64/amd64/' -e 's/\(arm\)\(64\)\?.*/\1\2/' -e 's/aarch64$/arm64/')" &&
  KREW="krew-${OS}_${ARCH}" &&
  curl -fsSLO "https://github.com/kubernetes-sigs/krew/releases/latest/download/${KREW}.tar.gz" &&
  tar zxvf "${KREW}.tar.gz" &&
  ./"${KREW}" install krew
)
export PATH="${KREW_ROOT:-$HOME/.krew}/bin:$PATH"

kubectl krew install neat          # strip managed fields and status from `get -o yaml`
kubectl krew install tree          # show owner-reference tree of an object
kubectl krew install view-secret   # decode a secret key without a jsonpath dance
kubectl krew install resource-capacity  # requests/limits vs allocatable per node
kubectl krew install who-can       # reverse RBAC lookup
kubectl krew install get-all       # really get everything in a namespace, including CRs
```

---

## 4. The Capstone kind Cluster

### 4.1 Why this shape

| Choice | Reason |
|---|---|
| 1 control-plane + 2 workers | Lets you observe scheduling, anti-affinity, topology spread, node drain, and taints. Single-node clusters hide half of Kubernetes |
| `disableDefaultCNI: true` + Cilium | kind's default CNI (kindnet) does **not** enforce NetworkPolicy. Cilium does, replaces kube-proxy with eBPF, and provides Hubble flow logs for the NetworkPolicy troubleshooting labs |
| `kubeProxyMode: none` | Cilium takes over Service load balancing; mirrors a modern EKS setup |
| Port mappings 80/443 on a worker | Gateway API and Ingress controllers receive traffic from `localhost` on your laptop without `port-forward` |
| Local registry on `localhost:5001` | Build images locally, push in a second, no rate limits. Mirrors how CI pushes to GHCR/ECR |
| Node labels `topology.kubernetes.io/zone` | Simulate multi-AZ for topology spread constraints and PDB labs |
| Pinned node image digest | Reproducibility across teammates and CI |
| Feature gates and audit logging | Lets you inspect API server audit events in the security labs |

### 4.2 `kind-config.yaml`

Save as `deploy/kind/kind-config.yaml` in the capstone repo.

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: capstone
networking:
  disableDefaultCNI: true        # we install Cilium
  kubeProxyMode: none            # Cilium eBPF replaces kube-proxy
  podSubnet: 10.244.0.0/16
  serviceSubnet: 10.96.0.0/12
  apiServerAddress: 127.0.0.1
  apiServerPort: 6443
containerdConfigPatches:
  - |-
    [plugins."io.containerd.grpc.v1.cri".registry]
      config_path = "/etc/containerd/certs.d"
kubeadmConfigPatches:
  - |
    kind: ClusterConfiguration
    apiServer:
      extraArgs:
        audit-log-path: /var/log/kubernetes/kube-apiserver-audit.log
        audit-policy-file: /etc/kubernetes/policies/audit-policy.yaml
        audit-log-maxage: "7"
        audit-log-maxbackup: "2"
        audit-log-maxsize: "100"
      extraVolumes:
        - name: audit-policies
          hostPath: /etc/kubernetes/policies
          mountPath: /etc/kubernetes/policies
          readOnly: true
          pathType: DirectoryOrCreate
        - name: audit-logs
          hostPath: /var/log/kubernetes
          mountPath: /var/log/kubernetes
          readOnly: false
          pathType: DirectoryOrCreate
nodes:
  - role: control-plane
    image: kindest/node:v1.31.4
    extraMounts:
      - hostPath: ./audit-policy.yaml
        containerPath: /etc/kubernetes/policies/audit-policy.yaml
        readOnly: true
    labels:
      topology.kubernetes.io/region: local
      topology.kubernetes.io/zone: local-a
  - role: worker
    image: kindest/node:v1.31.4
    labels:
      topology.kubernetes.io/region: local
      topology.kubernetes.io/zone: local-a
      node-role.kubernetes.io/worker: ""
      workload-tier: general
    extraPortMappings:
      - containerPort: 30080      # Gateway/Ingress NodePort for HTTP
        hostPort: 80
        protocol: TCP
      - containerPort: 30443      # Gateway/Ingress NodePort for HTTPS
        hostPort: 443
        protocol: TCP
  - role: worker
    image: kindest/node:v1.31.4
    labels:
      topology.kubernetes.io/region: local
      topology.kubernetes.io/zone: local-b
      node-role.kubernetes.io/worker: ""
      workload-tier: general
```

A minimal audit policy, saved next to it as `deploy/kind/audit-policy.yaml`:

```yaml
apiVersion: audit.k8s.io/v1
kind: Policy
omitStages: ["RequestReceived"]
rules:
  # Do not log read-only requests from system components
  - level: None
    users: ["system:kube-proxy", "system:kube-scheduler", "system:kube-controller-manager"]
    verbs: ["get", "list", "watch"]
  # Secrets: log metadata only, never the payload
  - level: Metadata
    resources:
      - group: ""
        resources: ["secrets", "configmaps"]
  # Everything mutating at RequestResponse level
  - level: RequestResponse
    verbs: ["create", "update", "patch", "delete", "deletecollection"]
  # Default
  - level: Metadata
```

### 4.3 The bootstrap script

Save as `deploy/kind/up.sh`. It is idempotent: re-running it on an existing cluster only reapplies add-ons.

```bash
#!/usr/bin/env bash
set -euo pipefail

CLUSTER_NAME=capstone
REG_NAME=kind-registry
REG_PORT=5001
CILIUM_VERSION=1.16.4
GATEWAY_API_VERSION=v1.2.0
ENVOY_GATEWAY_VERSION=v1.2.1
METRICS_SERVER_VERSION=3.12.2
HERE="$(cd "$(dirname "$0")" && pwd)"

echo "==> 1/7 Local registry"
if [ "$(docker inspect -f '{{.State.Running}}' "${REG_NAME}" 2>/dev/null || true)" != 'true' ]; then
  docker run -d --restart=always -p "127.0.0.1:${REG_PORT}:5000" \
    --network bridge --name "${REG_NAME}" registry:2
fi

echo "==> 2/7 kind cluster"
if ! kind get clusters | grep -qx "${CLUSTER_NAME}"; then
  (cd "${HERE}" && kind create cluster --config kind-config.yaml --wait 0s)
fi
kubectl config use-context "kind-${CLUSTER_NAME}"

echo "==> 3/7 Registry wiring (containerd hosts.toml on every node)"
REGISTRY_DIR="/etc/containerd/certs.d/localhost:${REG_PORT}"
for node in $(kind get nodes --name "${CLUSTER_NAME}"); do
  docker exec "${node}" mkdir -p "${REGISTRY_DIR}"
  cat <<EOF | docker exec -i "${node}" cp /dev/stdin "${REGISTRY_DIR}/hosts.toml"
[host."http://${REG_NAME}:5000"]
EOF
done
if [ "$(docker inspect -f='{{json .NetworkSettings.Networks.kind}}' "${REG_NAME}")" = 'null' ]; then
  docker network connect kind "${REG_NAME}"
fi
kubectl apply -f - <<EOF
apiVersion: v1
kind: ConfigMap
metadata:
  name: local-registry-hosting
  namespace: kube-public
data:
  localRegistryHosting.v1: |
    host: "localhost:${REG_PORT}"
    help: "https://kind.sigs.k8s.io/docs/user/local-registry/"
EOF

echo "==> 4/7 Gateway API CRDs (standard channel)"
kubectl apply -f "https://github.com/kubernetes-sigs/gateway-api/releases/download/${GATEWAY_API_VERSION}/standard-install.yaml"

echo "==> 5/7 Cilium (kube-proxy replacement, Hubble, Gateway API support)"
helm repo add cilium https://helm.cilium.io >/dev/null
helm repo update >/dev/null
helm upgrade --install cilium cilium/cilium --version "${CILIUM_VERSION}" \
  --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost="${CLUSTER_NAME}-control-plane" \
  --set k8sServicePort=6443 \
  --set ipam.mode=kubernetes \
  --set hubble.enabled=true \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true \
  --set gatewayAPI.enabled=false \
  --set operator.replicas=1 \
  --wait
cilium status --wait

echo "==> 6/7 metrics-server (kind kubelets use self-signed certs)"
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/ >/dev/null
helm upgrade --install metrics-server metrics-server/metrics-server \
  --version "${METRICS_SERVER_VERSION}" --namespace kube-system \
  --set args="{--kubelet-insecure-tls,--kubelet-preferred-address-types=InternalIP}" \
  --wait

echo "==> 7/7 Envoy Gateway (Gateway API implementation)"
helm upgrade --install eg oci://docker.io/envoyproxy/gateway-helm --version "${ENVOY_GATEWAY_VERSION}" \
  --namespace envoy-gateway-system --create-namespace --wait
kubectl wait --timeout=5m -n envoy-gateway-system deployment/envoy-gateway --for=condition=Available

echo "==> Done. Context: $(kubectl config current-context)"
kubectl get nodes -o wide
```

Run it:

```bash
chmod +x deploy/kind/up.sh
./deploy/kind/up.sh
```

Expected tail of the output (around four minutes on a laptop):

```text
==> Done. Context: kind-capstone
NAME                     STATUS   ROLES           AGE     VERSION   INTERNAL-IP   OS-IMAGE                         KERNEL-VERSION     CONTAINER-RUNTIME
capstone-control-plane   Ready    control-plane   3m50s   v1.31.4   172.18.0.4    Debian GNU/Linux 12 (bookworm)   6.6.36-linuxkit    containerd://1.7.18
capstone-worker          Ready    <none>          3m30s   v1.31.4   172.18.0.2    Debian GNU/Linux 12 (bookworm)   6.6.36-linuxkit    containerd://1.7.18
capstone-worker2         Ready    <none>          3m30s   v1.31.4   172.18.0.3    Debian GNU/Linux 12 (bookworm)   6.6.36-linuxkit    containerd://1.7.18
```

Nodes stay `NotReady` until Cilium is up; this is expected because we disabled the default CNI.

### 4.4 Expose the Gateway on localhost

Envoy Gateway creates a `LoadBalancer` Service per `Gateway`. kind has no cloud load balancer, so we patch the generated Service to use the fixed NodePorts mapped in `kind-config.yaml`. This is done once in Lab 4 after the first `Gateway` exists; a one-liner is kept here for reference:

```bash
# After creating a Gateway named "capstone-gw" in namespace "capstone":
SVC=$(kubectl get svc -n envoy-gateway-system -l gateway.envoyproxy.io/owning-gateway-name=capstone-gw -o name)
kubectl patch -n envoy-gateway-system "${SVC}" --type=json -p='[
  {"op":"replace","path":"/spec/type","value":"NodePort"},
  {"op":"replace","path":"/spec/ports/0/nodePort","value":30080}
]'
```

Alternatively, install `cloud-provider-kind` which emulates a LoadBalancer for kind clusters:

```bash
go install sigs.k8s.io/cloud-provider-kind@latest
sudo cloud-provider-kind   # runs in the foreground; LoadBalancer Services get an EXTERNAL-IP
```

### 4.5 Makefile targets

```makefile
.PHONY: cluster-up cluster-down cluster-reset registry-push
cluster-up:
	./deploy/kind/up.sh
cluster-down:
	kind delete cluster --name capstone
	docker rm -f kind-registry || true
cluster-reset: cluster-down cluster-up
registry-push:   # make registry-push IMG=fastapi-svc TAG=1.0.0
	docker tag $(IMG):$(TAG) localhost:5001/$(IMG):$(TAG)
	docker push localhost:5001/$(IMG):$(TAG)
```

---

## 5. k3d Alternative (Low-Memory Laptops)

k3d runs k3s, which bundles flannel, Traefik, local-path-provisioner, and a servicelb. It uses about half the memory of kind. Trade-offs: flannel does not enforce NetworkPolicy (k3s ships its own kube-router based policy controller which does), Traefik occupies ports 80/443 (disable it if you want Envoy Gateway), and some conformance differences exist (sqlite instead of etcd by default in single-server mode).

```bash
k3d cluster create capstone \
  --servers 1 --agents 2 \
  --image rancher/k3s:v1.31.4-k3s1 \
  --port "80:80@loadbalancer" --port "443:443@loadbalancer" \
  --k3s-arg "--disable=traefik@server:0" \
  --k3s-arg "--flannel-backend=none@server:0" \
  --k3s-arg "--disable-network-policy@server:0" \
  --k3s-arg "--disable-kube-proxy@server:0" \
  --registry-create capstone-registry:0.0.0.0:5001 \
  --wait

# Then install Cilium exactly as in up.sh, with k8sServiceHost set to the k3d server container:
helm upgrade --install cilium cilium/cilium --version 1.16.4 --namespace kube-system \
  --set kubeProxyReplacement=true \
  --set k8sServiceHost=k3d-capstone-server-0 --set k8sServicePort=6443 \
  --set ipam.mode=kubernetes --set hubble.relay.enabled=true --set hubble.ui.enabled=true --wait
```

All labs work on k3d with these flags. Where behaviour differs it is noted.

---

## 5a. Bare-Metal / VMware Alternative (Rocky Linux Multi-VM Cluster)

If you need a true multi-virtual-machine cluster that closely mirrors physical bare-metal or on-premises enterprise platforms (distinct kernels, dedicated systemd init subsystems, cgroup v2, and realistic node network interfaces) rather than container-in-container abstraction:

Follow our dedicated guide: **[01a-rocky-linux-vmware-setup.md](01a-rocky-linux-vmware-setup.md)** for a 3-Node Cluster (1 control plane, 2 workers) on Rocky Linux 9 using VMware Workstation / ESXi with CRI-O, kubeadm, and Calico CNI.

---

## 6. kubeconfig and Contexts

`kubectl` reads `~/.kube/config` (or the files in `$KUBECONFIG`). The file has three lists and one pointer:

```yaml
apiVersion: v1
kind: Config
clusters:                       # WHERE: API server endpoint + CA
  - name: kind-capstone
    cluster:
      server: https://127.0.0.1:6443
      certificate-authority-data: <BASE64_CA_CERT>
users:                          # WHO: credentials or an exec plugin that fetches them
  - name: kind-capstone
    user:
      client-certificate-data: <BASE64_CLIENT_CERT>   # kind uses client certs
      client-key-data: <BASE64_CLIENT_KEY>
  - name: eks-capstone
    user:
      exec:                      # EKS uses short-lived tokens from the AWS CLI
        apiVersion: client.authentication.k8s.io/v1beta1
        command: aws
        args: ["eks", "get-token", "--cluster-name", "capstone", "--region", "<REGION>"]
        env:
          - name: AWS_PROFILE
            value: capstone-dev
contexts:                       # WHERE + WHO + default namespace
  - name: kind-capstone
    context: { cluster: kind-capstone, user: kind-capstone, namespace: capstone }
  - name: eks-capstone
    context: { cluster: eks-capstone, user: eks-capstone, namespace: capstone }
current-context: kind-capstone
```

Daily commands:

```bash
kubectl config get-contexts                        # list, * marks current
kubectl config current-context
kubectl config use-context kind-capstone           # switch cluster
kubectl config set-context --current --namespace=capstone   # default namespace for this context
kubectx                                            # interactive / fuzzy switch
kubens capstone                                    # switch namespace
kubectl config view --minify --raw                 # dump only the current context's config (with secrets)
kubectl config view --flatten > merged.yaml        # merge $KUBECONFIG files into one portable file
kubectl --context eks-capstone get nodes           # one-off without switching
```

Security notes: kubeconfig files for kind contain a client certificate with `system:masters` (cluster-admin). Never commit them. Treat `~/.kube` as `chmod 700`. For EKS, never embed static tokens; the `exec` plugin calls `aws eks get-token` which signs a presigned STS `GetCallerIdentity` request valid for 15 minutes using your SSO session.

---

## 7. Amazon EKS Access

The EKS cluster is created by Terraform in cluster 05. This section covers only how to connect and what differs from kind.

### 7.1 Authenticate

```bash
aws sso login --profile capstone-dev
aws eks update-kubeconfig --name capstone --region <REGION> --profile capstone-dev --alias eks-capstone
kubectl --context eks-capstone get nodes
```

`update-kubeconfig` writes the `exec` stanza shown above. The token presented to the API server is validated by the EKS authenticator, which maps your IAM principal to a Kubernetes user and groups through **EKS access entries** (the modern replacement for the `aws-auth` ConfigMap):

```bash
# Who am I in Kubernetes terms?
kubectl --context eks-capstone auth whoami

# List access entries (IAM principals allowed into the cluster) and their policies
aws eks list-access-entries --cluster-name capstone --region <REGION>
aws eks list-associated-access-policies --cluster-name capstone --region <REGION> \
  --principal-arn arn:aws:iam::123456789012:role/CapstoneDeveloper
```

Grant a role read-only access without touching RBAC YAML:

```bash
aws eks create-access-entry --cluster-name capstone --region <REGION> \
  --principal-arn arn:aws:iam::123456789012:role/CapstoneViewer --type STANDARD
aws eks associate-access-policy --cluster-name capstone --region <REGION> \
  --principal-arn arn:aws:iam::123456789012:role/CapstoneViewer \
  --policy-arn arn:aws:eks::aws:cluster-access-policy/AmazonEKSViewPolicy \
  --access-scope type=namespace,namespaces=capstone
```

### 7.2 What differs from kind

| Area | kind | EKS |
|---|---|---|
| Control plane | One container, you can `docker exec` into it and read etcd | Fully managed, no access to etcd or control-plane nodes; logs via CloudWatch (`aws eks update-cluster-config --logging`) |
| CNI | Cilium (overlay/VXLAN) | Amazon VPC CNI (Pods get real VPC IPs; ENI/IP limits per instance type) optionally with Cilium chained or in ENI mode |
| kube-proxy | Replaced by Cilium | Managed add-on, iptables mode by default |
| Storage | No CSI by default (kind ships `standard` StorageClass via local-path-provisioner) | EBS CSI driver add-on (`gp3`), EFS CSI for RWX |
| LoadBalancer Services | None (NodePort or cloud-provider-kind) | AWS Load Balancer Controller creates NLB/ALB; Gateway API via Envoy Gateway or the AWS controller |
| Node identity for Pods | None | IRSA (OIDC federation on ServiceAccount tokens) or EKS Pod Identity |
| Authentication | Client certificate | IAM via `aws eks get-token`, mapped by access entries |
| Autoscaling | Static nodes | Karpenter (recommended) or Cluster Autoscaler on managed node groups |
| Cost | Zero | Roughly 0.10 USD/hour for the control plane plus nodes; destroy nightly |

### 7.3 Verify the EKS add-ons

```bash
aws eks list-addons --cluster-name capstone --region <REGION>
kubectl --context eks-capstone get pods -n kube-system
kubectl --context eks-capstone get storageclass
kubectl --context eks-capstone get daemonset -n kube-system aws-node   # VPC CNI
```

---

## 8. Verification Checklist

Run this after `up.sh` and before every lab session. Each command has a one-line reason.

```bash
# 1. API server reachable and version matches expectation
kubectl version
kubectl cluster-info

# 2. All nodes Ready, correct labels
kubectl get nodes --show-labels

# 3. All system Pods Running (Cilium, CoreDNS, metrics-server, Envoy Gateway)
kubectl get pods --all-namespaces -o wide

# 4. Cilium healthy and kube-proxy replacement active
cilium status
kubectl -n kube-system exec ds/cilium -- cilium-dbg status --brief

# 5. metrics-server serving (needed for HPA and kubectl top)
kubectl top nodes
kubectl get apiservice v1beta1.metrics.k8s.io -o jsonpath='{.status.conditions[?(@.type=="Available")].status}{"\n"}'

# 6. Gateway API CRDs installed and a GatewayClass accepted
kubectl get crd gateways.gateway.networking.k8s.io
kubectl get gatewayclass

# 7. DNS resolves inside the cluster
kubectl run dns-test --rm -it --restart=Never --image=busybox:1.36 -- nslookup kubernetes.default.svc.cluster.local

# 8. Local registry reachable from nodes
docker pull busybox:1.36 && docker tag busybox:1.36 localhost:5001/busybox:1.36 && docker push localhost:5001/busybox:1.36
kubectl run reg-test --rm -it --restart=Never --image=localhost:5001/busybox:1.36 -- echo "registry OK"

# 9. Default StorageClass exists (kind ships "standard")
kubectl get storageclass

# 10. Audit log is being written on the control plane
docker exec capstone-control-plane tail -n 2 /var/log/kubernetes/kube-apiserver-audit.log
```

Expected highlights:

```text
$ cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             OK
 \__/¯¯\__/    Operator:           OK
 /¯¯\__/¯¯\    Envoy DaemonSet:    OK
 \__/¯¯\__/    Hubble Relay:       OK
    \__/       ClusterMesh:        disabled
DaemonSet              cilium             Desired: 3, Ready: 3/3, Available: 3/3
Deployment             cilium-operator    Desired: 1, Ready: 1/1, Available: 1/1
KubeProxyReplacement:  True

$ kubectl top nodes
NAME                     CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
capstone-control-plane   312m         7%     1021Mi          13%
capstone-worker          98m          2%     512Mi           6%
capstone-worker2         91m          2%     498Mi           6%

$ kubectl get gatewayclass
NAME   CONTROLLER                                      ACCEPTED   AGE
eg     gateway.envoyproxy.io/gatewayclass-controller   True       2m
```

If `GatewayClass eg` is missing, create it; Envoy Gateway does not create one by default:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

```bash
kubectl apply -f deploy/kind/gatewayclass.yaml
```

Run the Cilium connectivity test once (takes several minutes, creates and destroys its own namespace):

```bash
cilium connectivity test --test '!pod-to-world' --test '!to-fqdns'
```

---

## 9. First-Run Configuration: Namespaces and Guardrails

Create the capstone namespace with Pod Security Admission labels so every later lab is forced to write compliant manifests from day one:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: capstone
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.31
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/audit: restricted
    environment: dev
    team: platform
```

```bash
kubectl apply -f deploy/base/namespace.yaml
kubectl config set-context --current --namespace=capstone
kubectl get ns capstone --show-labels
```

Test that the guardrail bites. A naive Pod must be rejected:

```bash
kubectl run naughty --image=nginx:1.27 --restart=Never
```

Expected output:

```text
Error from server (Forbidden): pods "naughty" is forbidden: violates PodSecurity "restricted:v1.31":
allowPrivilegeEscalation != false (container "naughty" must set securityContext.allowPrivilegeEscalation=false),
unrestricted capabilities (container "naughty" must set securityContext.capabilities.drop=["ALL"]),
runAsNonRoot != true (pod or container "naughty" must set securityContext.runAsNonRoot=true),
seccompProfile (pod or container "naughty" must set securityContext.seccompProfile.type to "RuntimeDefault" or "Localhost")
```

That error message is your checklist for every Pod spec in this folder. [02e-security.md](02e-security.md) explains each line.

---

## 10. Teardown and Reset

```bash
# Delete everything, including the registry (images are lost)
make cluster-down

# Keep the registry (image cache) but recreate the cluster: about 4 minutes
kind delete cluster --name capstone && ./deploy/kind/up.sh

# Free Docker disk space taken by old kind node images
docker image prune --filter "label=io.x-k8s.kind.cluster" --force
docker system df
```

For EKS, never leave the cluster running overnight. The capstone repository includes a scheduled GitHub Actions workflow (`.github/workflows/nightly-destroy.yml`, cluster 06) that runs `terraform destroy -auto-approve` on the `infra/terraform/eks` stack at 23:00 UTC. The rule from the roadmap is to keep lab cloud spend under 15 USD/month; one forgotten EKS cluster with two `m6i.large` nodes costs roughly 10 USD per day.

---

## 11. Common Setup Problems

| Symptom | Cause | Fix |
|---|---|---|
| `kind create cluster` hangs at "Starting control-plane" | Docker has too little memory, or inotify limits exhausted on Linux | Give Docker 6 GB+; on Linux `sudo sysctl fs.inotify.max_user_watches=524288 fs.inotify.max_user_instances=512` |
| Nodes stay `NotReady` forever | CNI not installed (we disabled kindnet) | Run the Cilium step; `kubectl -n kube-system get pods -l k8s-app=cilium` |
| `cilium status` shows `KubeProxyReplacement: False` | `k8sServiceHost` wrong, Cilium cannot reach the API server without kube-proxy | Set `--set k8sServiceHost=capstone-control-plane --set k8sServicePort=6443` and reinstall |
| `kubectl top nodes` says `Metrics API not available` | metrics-server cannot verify kubelet certs | Add `--kubelet-insecure-tls` arg (kind only; never on EKS) |
| `ImagePullBackOff` for `localhost:5001/...` | containerd `hosts.toml` not written or registry not on the `kind` Docker network | Re-run step 3 of `up.sh`; `docker network inspect kind` should list `kind-registry` |
| `connection refused` to `127.0.0.1:6443` after laptop sleep | kind containers stopped | `docker start capstone-control-plane capstone-worker capstone-worker2` |
| Windows: `kind` works but Cilium Pods crash with eBPF mount errors | Docker Desktop not using WSL2 backend | Enable WSL2 integration in Docker Desktop settings |
| `aws eks get-token` returns `ExpiredToken` | SSO session expired | `aws sso login --profile capstone-dev` |
| `error: You must be logged in to the server (Unauthorized)` on EKS | IAM principal has no access entry | Create an access entry (section 7.1); the cluster creator is the only default principal |

Deeper diagnostics for everything that can go wrong after setup live in [06-troubleshooting.md](06-troubleshooting.md).

---

## Next

Proceed to [02-core-concepts.md](02-core-concepts.md). Keep the cluster running; every concept there is accompanied by a command you can execute against it.

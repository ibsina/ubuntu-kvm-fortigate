# FortiAIGate: HTTP Docker Registry with Kubernetes containerd

This guide configures **standard Kubernetes using containerd** (for example, a kubeadm cluster) to pull FortiAIGate images from a private **HTTP** Docker Registry.

## Lab details

| Setting | Value |
|---|---|
| Registry | `http://58.185.5.124` (port 80) |
| Registry API | `http://58.185.5.124/v2/` |
| Kubernetes image prefix | `58.185.5.124/` (no `http://`) |
| Example image | `58.185.5.124/webui:build0055` |
| Container runtime | containerd 2.x in the verified lab |
| CRI registry config directory | `/etc/containerd/certs.d` |

> **Security:** HTTP transfers are unencrypted. Restrict registry access to trusted lab nodes, protect push operations with authentication/authorization, and prefer HTTPS with a trusted certificate for production. If you use a hostname or a non-default port instead of this IP, replace the registry host consistently in the directory name, `hosts.toml`, and image references.

## 1. Identify the runtime on each node

```bash
kubectl get nodes -o wide
kubectl get pods -n fortiaigate -o wide
containerd --version
sudo grep -n -A 2 -B 1 'registry]' /etc/containerd/config.toml
```

In the verified **containerd 2.x** lab, the relevant configuration is:

```toml
[plugins.'io.containerd.cri.v1.images'.registry]
  config_path = '/etc/containerd/certs.d'
```

For **containerd 1.x**, the equivalent CRI section is usually:

```toml
[plugins."io.containerd.grpc.v1.cri".registry]
  config_path = "/etc/containerd/certs.d"
```

Do not overwrite an existing `config.toml` just to add this registry. Confirm the active CRI `config_path` first. If the config path is already set, adding a `hosts.toml` generally does not require a containerd restart.

## 2. Configure the HTTP registry

Run **on every Kubernetes node that may schedule FortiAIGate pods**, including GPU workers:

```bash
sudo mkdir -p /etc/containerd/certs.d/58.185.5.124

sudo tee /etc/containerd/certs.d/58.185.5.124/hosts.toml >/dev/null <<'EOF_HOSTS'
server = "http://58.185.5.124"

[host."http://58.185.5.124"]
  capabilities = ["pull", "resolve"]
EOF_HOSTS

sudo cat /etc/containerd/certs.d/58.185.5.124/hosts.toml
```

Do not use `http://` inside Kubernetes image names.

## 3. Verify registry access

```bash
curl -i http://58.185.5.124/v2/
curl -fsS http://58.185.5.124/v2/webui/tags/list
```

For an unauthenticated registry, `/v2/` normally returns HTTP 200 with the `Docker-Distribution-Api-Version: registry/2.0` header. A registry with authentication may return HTTP 401 with an authentication challenge. The tag list must include `build0055` before you attempt to pull that image.

## 4. Verify the Kubernetes CRI pull

Run on the **node that will actually run the pod**:

```bash
sudo crictl \
  --runtime-endpoint unix:///run/containerd/containerd.sock \
  --image-endpoint unix:///run/containerd/containerd.sock \
  pull 58.185.5.124/webui:build0055
```

If `crictl` is not installed, install a version compatible with your Kubernetes runtime, or test with a small Kubernetes pod and inspect its events. A successful `ctr images pull --plain-http` alone does **not** prove Kubernetes CRI pulls are configured correctly.

## 5. Set the image in Helm values

The exact Helm keys depend on the FortiAIGate chart. Example only:

```yaml
image:
  repository: 58.185.5.124/webui
  tag: build0055
```

The image reference is **`58.185.5.124/webui:build0055`**, without a URL scheme. Verify the chart's actual image fields and init-container images before deploying.

### Image tags uploaded in this lab

| Repository | Tag |
|---|---|
| `api` | `build0055` |
| `guardrail-scanner` | `build0055` |
| `guardrail` | `build0055` |
| `license_manager` | `build0055` |
| `llm-gateway` | `build0055` |
| `logd` | `build0055` |
| `webui` | `build0055` |
| `custom-triton` | `25.11-onnx-trt-agt-rpt-s1` |
| `faig-models` | `0.0.5-s1` |
| `vllm-server` | `0.28.0-rpt-s1` |

These tags assume you pushed the images using the earlier lab's tagging convention. Check the registry's actual tag list and the Helm chart before use.

## 6. Troubleshooting

Inspect pods and events:

```bash
kubectl get pods -n fortiaigate -o wide
kubectl get events -n fortiaigate --sort-by=.lastTimestamp | tail -40
kubectl describe pod -n fortiaigate <POD_NAME>
```

- **`Head "https://58.185.5.124/...": x509 ... IP SANs`**: containerd is still attempting HTTPS. Confirm `hosts.toml` and the active CRI `config_path` **on the node listed in the pod's `NODE` field**. In the previous lab, configuring only `faig-ctl` did not fix a pod scheduled on `faig-node1`.
- **`http: server gave HTTP response to HTTPS client`**: same likely issue—HTTP registry config not being applied on the pulling node.
- **`manifest unknown` / `not found`**: check the exact repository and tag via `/v2/<repo>/tags/list`.
- **`connection refused` / timeout**: verify routing, firewall, registry listener, and any reverse proxy.
- **`401 Unauthorized`**: configure the registry credentials and, if required, Kubernetes `imagePullSecrets`.

Only restart containerd if you changed the active main runtime configuration or have confirmed that the registry hosts configuration is not being picked up. Restarting containerd can disrupt workloads; plan changes per node.

## Optional: Check the Docker Registry API

```bash
for repo in api webui custom-triton faig-models vllm-server; do
  echo "=== $repo ==="
  curl -fsS "http://58.185.5.124/v2/${repo}/tags/list"
  echo
done
```

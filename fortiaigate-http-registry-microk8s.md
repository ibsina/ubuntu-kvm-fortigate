# FortiAIGate: HTTP Docker Registry with MicroK8s

This guide configures **MicroK8s on Ubuntu** to pull FortiAIGate build0055 images from an existing private **HTTP** Docker Registry.

## Lab details

| Setting | Value |
|---|---|
| Registry | `http://58.185.5.124` (port 80) |
| Registry API | `http://58.185.5.124/v2/` |
| Kubernetes image prefix | `58.185.5.124/` (no `http://`) |
| Example image | `58.185.5.124/webui:build0055` |
| Runtime | MicroK8s-managed containerd |
| Registry hosts directory | `/var/snap/microk8s/current/args/certs.d` |

> **Security:** HTTP transfers are unencrypted. Limit access to trusted lab nodes and protect registry pushes. Prefer HTTPS with a trusted certificate for production. This guide targets MicroK8s **1.23+**, where registry `hosts.toml` is supported; verify your installed version and active configuration.

## 1. Verify MicroK8s

```bash
microk8s version
microk8s status --wait-ready
microk8s kubectl get nodes -o wide
```

For a multi-node cluster, configure **every MicroK8s node that may pull images**, including GPU workers.

## 2. Configure the HTTP registry

On each relevant MicroK8s node:

```bash
sudo mkdir -p /var/snap/microk8s/current/args/certs.d/58.185.5.124

sudo tee /var/snap/microk8s/current/args/certs.d/58.185.5.124/hosts.toml >/dev/null <<'EOF_HOSTS'
server = "http://58.185.5.124"

[host."http://58.185.5.124"]
  capabilities = ["pull", "resolve"]
EOF_HOSTS

sudo cat /var/snap/microk8s/current/args/certs.d/58.185.5.124/hosts.toml
```

Check MicroK8s' containerd template for the registry hosts path:

```bash
sudo grep -n 'config_path' /var/snap/microk8s/current/args/containerd-template.toml
```

If your version or template uses a different path, follow the actual MicroK8s configuration; do not edit standard `/etc/containerd/config.toml` for a MicroK8s-managed runtime.

### Applying changes

For a disposable **single-node lab**, you can restart MicroK8s if the registry configuration is not recognized:

```bash
microk8s stop
microk8s start
microk8s status --wait-ready
```

This interrupts running workloads. On a multi-node cluster, plan node-by-node changes rather than stopping the entire cluster. Adding a hosts file may be picked up without a restart depending on the runtime/version; test first.

## 3. Verify registry connectivity

```bash
curl -i http://58.185.5.124/v2/
curl -fsS http://58.185.5.124/v2/webui/tags/list
```

For an unauthenticated registry, `/v2/` should normally return HTTP 200 and `Docker-Distribution-Api-Version: registry/2.0`. If the registry requires authentication, HTTP 401 with a valid challenge may be expected. Confirm the `build0055` tag exists.

## 4. Verify image pulling

### Optional: low-level containerd test

```bash
sudo microk8s ctr images pull --plain-http 58.185.5.124/webui:build0055
```

This confirms the registry can be reached using an explicitly HTTP-enabled `ctr` request; **it does not verify that Kubernetes CRI image pulling works**.

### Required: CRI test

If your MicroK8s installation provides the wrapper:

```bash
sudo microk8s crictl pull 58.185.5.124/webui:build0055
```

If the wrapper is unavailable, check the MicroK8s socket and use a compatible standalone `crictl`:

```bash
sudo ls -l /var/snap/microk8s/common/run/containerd.sock
sudo crictl \
  --runtime-endpoint unix:///var/snap/microk8s/common/run/containerd.sock \
  --image-endpoint unix:///var/snap/microk8s/common/run/containerd.sock \
  pull 58.185.5.124/webui:build0055
```

A successful CRI pull confirms the MicroK8s runtime can pull the image without requiring `--plain-http` on each pull.

## 5. Configure FortiAIGate Helm values

Use the image repository **without** `http://`. Example only; actual keys depend on the build0055 Helm chart:

```yaml
image:
  repository: 58.185.5.124/webui
  tag: build0055
```

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

These names reflect the earlier lab's push convention; verify the actual tags and chart references.

If Helm is installed as the MicroK8s addon, use `microk8s helm3` for Helm operations; otherwise, use your separately installed `helm` CLI with the correct Kubernetes context.

## 6. Verify pods and troubleshoot

```bash
microk8s kubectl get pods -n fortiaigate -o wide
microk8s kubectl get events -n fortiaigate --sort-by=.lastTimestamp | tail -40
microk8s kubectl describe pod -n fortiaigate <POD_NAME>
```

- **`Head "https://58.185.5.124/...": x509 ... IP SANs`**: the worker's containerd is still trying HTTPS. Check the MicroK8s `hosts.toml` **on the node where the pod is scheduled**.
- **`http: server gave HTTP response to HTTPS client`**: the runtime is not using the HTTP registry configuration.
- **`manifest unknown`**: check the repository/tag using `/v2/<repo>/tags/list`.
- **Timeout / connection refused**: verify routing, firewall, HTTP listener, and any FortiWeb reverse-proxy behavior.
- **`401 Unauthorized`**: registry credentials and possibly Kubernetes `imagePullSecrets` are required.

Avoid changing the Helm chart until the CRI pull succeeds on every relevant worker node.

## References

- [MicroK8s: Working with a private registry](https://microk8s.io/docs/registry-private)
- [containerd: Registry configuration (`hosts.toml`)](https://github.com/containerd/containerd/blob/main/docs/hosts.md)

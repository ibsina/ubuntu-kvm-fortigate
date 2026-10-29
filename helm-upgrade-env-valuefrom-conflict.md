# Fixing `helm upgrade` failure: `env[6].valueFrom may not be specified when value is not empty`

## Symptom

`helm upgrade` on the FortiAIGate chart fails with:

```text
Error: UPGRADE FAILED: cannot patch "triton-vllm-server" with kind Deployment:
Deployment.apps "triton-vllm-server" is invalid:
spec.template.spec.containers[1].env[6].valueFrom: Invalid value: "":
may not be specified when `value` is not empty
```

## Cause

A manual `kubectl set env` was previously applied to the `vllm` container to work around a CUDA/CDI problem:

```bash
kubectl -n fortiaigate set env deployment/triton-vllm-server -c vllm \
  NVIDIA_VISIBLE_DEVICES=nvidia.com/gpu=0
```

That turned the live `NVIDIA_VISIBLE_DEVICES` entry into a plain `value`. The chart defines the same variable with `valueFrom` (it reads the GPU UUID written by the `gpu-probe` init container). When Helm merges the chart's `valueFrom` into the live entry, the result has both `value` and `valueFrom`, which Kubernetes rejects.

The error points at `containers[1].env[6]`, which is the `vllm` container's `NVIDIA_VISIBLE_DEVICES`. The change being upgraded (for example a host value in `values.yaml`) is not the problem.

The separate `runtimeClassName: nvidia` patch does **not** conflict, because the chart does not define that field and Helm leaves it alone.

## Fix

Remove the manual env var, run the upgrade, then reapply the override. Pausing the rollout keeps it to a single pod restart.

```bash
kubectl -n fortiaigate rollout pause deploy/triton-vllm-server
kubectl -n fortiaigate set env deploy/triton-vllm-server -c vllm NVIDIA_VISIBLE_DEVICES-
helm upgrade fortiaigate ./fortiaigate -n fortiaigate
kubectl -n fortiaigate set env deploy/triton-vllm-server -c vllm NVIDIA_VISIBLE_DEVICES=nvidia.com/gpu=0
kubectl -n fortiaigate rollout resume deploy/triton-vllm-server
```

Notes:

- The trailing `-` in the second command deletes the variable. The upgrade then re-adds the chart's `valueFrom` version without a conflict.
- The later `set env` replaces it with the working `nvidia.com/gpu=0` value.
- Do not use `helm upgrade --wait` for this run. A paused deployment never becomes ready, so `--wait` would time out.

## Verify

```bash
kubectl -n fortiaigate get deploy triton-vllm-server \
  -o jsonpath='{.spec.template.spec.runtimeClassName}{"\n"}'
kubectl -n fortiaigate get pods -w | grep triton
POD=$(kubectl -n fortiaigate get pod -o name | grep triton-vllm-server | head -1)
kubectl -n fortiaigate exec $POD -c vllm -- nvidia-smi -L
```

Expected:

1. `runtimeClassName` prints `nvidia`.
2. The pod reaches `2/2 Running`.
3. `nvidia-smi -L` inside the `vllm` container lists the GPU.

If `runtimeClassName` comes back empty, reapply it:

```bash
kubectl -n fortiaigate patch deploy/triton-vllm-server --type=merge \
  -p '{"spec":{"template":{"spec":{"runtimeClassName":"nvidia"}}}}'
```

## Why this keeps happening

Every `helm upgrade` will hit this again while the env override is a manual patch, so use the same sequence each time. Permanent options:

- **Node-level:** make the NVIDIA runtime the containerd default for the cluster that Kubernetes actually uses, and adjust the toolkit's CDI default kind so a bare GPU UUID resolves under `nvidia.com/gpu`. Verify the exact setting for your toolkit version first.
- **Chart-level:** ask the FortiAIGate team for a `runtimeClassName` option and an env override for the `vllm` container. Shared mode currently assumes the NVIDIA runtime is containerd's default and that a bare UUID in `NVIDIA_VISIBLE_DEVICES` is accepted, which does not hold on a GPU Operator node in CDI mode with a vGPU profile.

## Related `values.yaml` cleanup

While in the file, in the `vllm:` block:

- Remove `sharedGpuDevices: triton`. The chart documents it as retired and refuses it at render time.
- Remove the duplicate `maxModelLen` key (it appears twice). Keep one.

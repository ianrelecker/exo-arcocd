# exo on Kubernetes

Kubernetes and Argo CD manifests for running [`exo`](https://github.com/exo-explore/exo) across your cluster. The default `ai/` Kustomize package deploys an `exo` DaemonSet (one pod per node), exposes it via a ClusterIP service, and includes an optional Ingress for HTTP access to port `52415`.

## Repository layout

- `ai/` – Base Kustomize package: creates the `exo` namespace, the DaemonSet, and the service.
- `ai-gpu/` – Overlay that swaps in the CUDA image, adds `CUDA=1`, and requests an NVIDIA GPU.
- `ai-tailscale/` – Overlay that adds a Tailscale sidecar for cross-node discovery.
- `demo/` – Single-pod Deployment variant (no hostPath volumes) for quick demos.
- `docker/` – Dockerfiles used to build CPU (`Dockerfile.cpu`) and NVIDIA (`Dockerfile.nvidia`) images.
- `app-exo.yaml` – Example Argo CD `Application` pointing at the `ai/` package.

## Building container images

The manifests expect images published under `ghcr.io/ianrelecker/exo`. Build and push the CPU (and optional NVIDIA) variants before syncing:

```bash
# CPU-only image (tinygrad via clang)
docker build -t ghcr.io/ianrelecker/exo:cpu -f docker/Dockerfile.cpu .
docker push ghcr.io/ianrelecker/exo:cpu

# Optional: NVIDIA GPU image
docker build -t ghcr.io/ianrelecker/exo:nvidia -f docker/Dockerfile.nvidia .
docker push ghcr.io/ianrelecker/exo:nvidia
```

Both images create a non-root `exo` user (UID 10001), clone the upstream source, and install it with `pip install -e .`. Runtime data lives at `/data`; the DaemonSet mounts host paths so downloaded models stay cached per node.

### Automated builds

This repository includes a GitHub Actions workflow (`.github/workflows/build-images.yaml`) that rebuilds both images and publishes them to GHCR whenever the `main` branch changes. It pushes two tags per build:

- `ghcr.io/ianrelecker/exo:cpu` and `:nvidia` (rolling tags you reference from Kubernetes)
- `ghcr.io/ianrelecker/exo:cpu-${GITHUB_SHA}` and `:nvidia-${GITHUB_SHA}` (immutable digests you can pin to)

You can also trigger the workflow manually from the Actions tab. If you fork this repository under a different account, no changes are required—the workflow automatically uses `github.repository_owner` for the GHCR namespace.

## Deploying with Kustomize

Apply the base package directly:

```bash
kubectl apply -k ai
```

Optional overlays:

```bash
kubectl apply -k ai-gpu        # NVIDIA nodes with nvidia.com/gpu available
kubectl apply -k ai-tailscale  # Adds Tailscale sidecar (requires ts-auth secret)
kubectl apply -k demo          # Single-pod deployment for quick tests
```

The workload invokes `exo` via `/usr/bin/env`, so whichever image you deploy just needs the CLI on the `PATH`. If you relocate the binary, update the container command in the manifests.

Create the Tailscale secret before using the overlay:

```bash
kubectl -n exo create secret generic ts-auth --from-literal=TS_AUTHKEY='tskey-xxxxx'
```

## Deploying with Argo CD

Update the repo URL, branch, or overlay path in `app-exo.yaml` if needed, then apply it in the Argo CD namespace:

```bash
kubectl apply -n argocd -f app-exo.yaml
```

To target a different overlay, copy the file and change `spec.source.path` accordingly (e.g., `ai-gpu`). Automated sync, prune, and self-heal are already enabled.

## Verifying the deployment

```bash
kubectl -n exo get ds exo -w        # or get deploy exo for the demo overlay
kubectl -n exo get svc exo          # ClusterIP service that fronts every pod
kubectl -n exo get ingress exo      # host/endpoint exposed by your ingress controller
```

Once your ingress controller announces the host (default `exo.local`, adjust as needed), open `http://<host>:52415` for the `exo` web UI. The Chat Completions API is served at `/v1/chat/completions` on the same endpoint.

## Notes & customization tips

- The DaemonSet mounts `/var/lib/exo-data` and `/var/lib/exo-cache` from each node. Adjust or swap to PVCs if you prefer shared storage.
- Pods run with `hostNetwork: true` so UDP discovery and TCP traffic originate from the node's real interface. Make sure port `52415/udp+tcp` is free on every node and permitted through any host firewall so laptops or other non-Kubernetes devices can auto-discover the cluster.
- GPU overlay assumes the NVIDIA device plugin is installed. Tweak resource requests/limits in `ai-gpu/gpu-patch.yaml` for your hardware.
- The Tailscale sidecar requires `NET_ADMIN` and `/dev/net/tun`; ensure your cluster policy allows it.
- Modify `app-exo.yaml` to change namespaces, projects, or sync settings to match your Argo CD conventions.

Happy exploring!

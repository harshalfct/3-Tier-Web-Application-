# GitHub Actions deployment setup

The workflow in `.github/workflows/ci-cd.yml` validates the Python and frontend projects, builds the backend and frontend images, publishes them to GitHub Container Registry (GHCR), and deploys immutable commit-tagged images to the `three-tier` namespace.

## Required GitHub configuration

Create a **production** environment and add these secrets:

- `KUBE_CONFIG`: kubeconfig for a dedicated, least-privilege deployment service account. Do not use the kubeadm admin kubeconfig.
- `GHCR_USERNAME`: a GitHub user or machine account that can read the two GHCR packages.
- `GHCR_TOKEN`: a fine-grained token with package read access. The workflow's `GITHUB_TOKEN` publishes the images; this token is used by Kubernetes nodes to pull private images.

The Kubernetes API endpoint in `KUBE_CONFIG` must be reachable from GitHub-hosted runners. If the API is private, run the deploy job on a self-hosted runner with the `self-hosted` label and change its `runs-on` value accordingly; do not expose the Kubernetes API publicly just for CI/CD.

## Cluster prerequisites

Install or configure these before the first deployment:

1. A default StorageClass or CSI driver for the MySQL PVC.
2. Either ingress-nginx for `k8s/07-ingress.yaml`, or use the existing NodePort proxy on port `30080`.
3. NetworkPolicy support in the cluster CNI.
4. DNS/network access from every node to `ghcr.io`.

The workflow creates the `three-tier` namespace and `ghcr-credentials` pull secret. The manifests still contain the database Secret template; replace the example credentials before production use, preferably by managing that Secret outside Git and removing `k8s/01-secret.yaml` from the deployment overlay.

## First-time checks

```bash
kubectl get nodes
kubectl get storageclass
kubectl get ingressclass
kubectl auth can-i apply -n three-tier --as=system:serviceaccount:ci:github-actions
```

After a successful run, the NodePort deployment is available at:

```text
http://<node-ip>:30080
```

Use `kubectl get pods,svc -n three-tier` and the workflow rollout logs to troubleshoot an unsuccessful deployment.

# Automated-CI-CD-Pipeline

## CI/CD

The GitHub Actions workflow in `.github/workflows/ci-cd.yml` lints the repository, builds and smoke-tests the demo container, and fails if Trivy finds fixable HIGH or CRITICAL vulnerabilities. Pull requests to `main` run these checks without publishing an image. Pushes to `main` publish the scanned image to GitHub Container Registry (GHCR). Kubernetes rollout is opt-in, so the demo pipeline can run without cluster credentials.

### Repository setup

- The included `Dockerfile` packages the static showcase page in an unprivileged Nginx image.
- To enable Kubernetes deployment on pushes to `main`, set the Actions variable `DEPLOY_ENABLED` to `true`.
- In repository **Settings > Secrets and variables > Actions**, add the `KUBE_CONFIG` secret containing the kubeconfig for the target cluster.
- Add these Actions variables: `K8S_DEPLOYMENT` (existing Deployment name), `K8S_CONTAINER` (container name in that Deployment), and `K8S_NAMESPACE`.
- Ensure the cluster can pull the GHCR image. For a private package, configure an image pull secret in the Deployment's namespace.
- Configure the `production` environment in repository settings if deployment approval or environment-specific protection is required.

The workflow deploys by changing the image on an existing Kubernetes Deployment; it does not create cluster resources. The container listens on port `8080`.

### Run the demo locally

```sh
docker build -t pipeline-demo:local .
docker run --rm -p 8080:8080 pipeline-demo:local
```

Open `http://localhost:8080` to view the pipeline showcase.

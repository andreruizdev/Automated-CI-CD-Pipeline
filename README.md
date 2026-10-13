# Automated-CI-CD-Pipeline

## CI/CD

The GitHub Actions workflow in `.github/workflows/ci-cd.yml` lints the repository,
builds and smoke-tests the demo container, scans it with Trivy, then deploys it
to a temporary `kind` cluster and tests the Service. It publishes to GitHub
Container Registry (GHCR) only after these checks pass. Pull requests to `main`
run the same checks without publishing. Deployment to a separate production
cluster is opt-in.

The `kind` deployment is an ephemeral CI integration test, not a production
deployment. The separate production job runs only on pushes to `main` when
`DEPLOY_ENABLED` is `true`; it is skipped in the default CI run.

### Repository setup

- The included `Dockerfile` packages the static showcase page in an unprivileged Nginx image.
- To enable deployment on pushes to `main`, set the Actions variable `DEPLOY_ENABLED` to `true`.
- In repository **Settings > Secrets and variables > Actions**, add the `KUBE_CONFIG` secret containing the kubeconfig for the target cluster. The identity needs permission to manage resources in the configured namespace.
- Ensure the cluster can pull the GHCR image. For a private package, configure an image pull secret in the Deployment's namespace.
- Configure the `production` environment in repository settings if deployment approval or environment-specific protection is required.
- The production overlay uses `ghcr.io/OWNER/REPOSITORY` as a generic image placeholder. Replace it when applying the overlay manually; GitHub Actions sets the deployed image to this repository's published commit digest.

The production overlay creates a namespace, Deployment, and ClusterIP Service.
Customize `k8s/base` for shared resources and `k8s/overlays/production` for
environment-specific image names, replica counts, or patches. The workflow
overrides the production image tag with the commit SHA. The demo container
listens on port `8080`.

### Test Kubernetes locally

Install Docker, `kubectl`, `kind`, and `jq`, then run:

```sh
docker build -t pipeline-demo:local .
kind create cluster --config k8s/kind/cluster.yaml
kind load docker-image pipeline-demo:local --name automated-ci-cd
kubectl apply -k k8s/overlays/local
kind_network_cidr="$(docker network inspect kind --format '{{json .IPAM.Config}}' | jq -r '.[] | select(.Subnet | contains(".")) | .Subnet')"
policy_patch="$(printf '[{\"op\":\"add\",\"path\":\"/spec/ingress/0/from/-\",\"value\":{\"ipBlock\":{\"cidr\":\"%s\"}}}]' "$kind_network_cidr")"
kubectl patch networkpolicy pipeline-demo-ingress --namespace pipeline-demo-local --type=json --patch "$policy_patch"
kubectl rollout status deployment/pipeline-demo --namespace pipeline-demo-local --timeout=120s
```

The local overlay exposes the app at `http://localhost:8081`. Clean up with `kind delete cluster --name automated-ci-cd`.

### Run the demo locally

```sh
docker build -t pipeline-demo:local .
docker run --rm -p 8080:8080 pipeline-demo:local
```

Open `http://localhost:8080` to view the pipeline showcase.

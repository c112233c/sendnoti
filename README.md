# sendnoti

Monorepo for the sendnoti notification gateway and its Kubernetes GitOps configuration.

## Repository layout

- `.github/workflows/build-be.yml` — backend CI, GHCR image build/publish, and GitOps image-tag update
- `sendnoti-be/` — Node.js backend, project-specific formatters, output adapters, tests, and Dockerfile
- `sendnoti-gitops/be/` — backend Helm chart consumed by Argo CD
- `sendnoti-gitops/argocd/sendnoti-be.yaml` — Argo CD Application manifest

## CI/CD flow

1. A push to `main` that changes `sendnoti-be/**` runs syntax checks, unit tests, and npm audit.
2. On success, GitHub Actions builds `sendnoti-be/Dockerfile` using `sendnoti-be/` as its build context.
3. The image is pushed to `ghcr.io/<github-owner>/sendnoti-be` with an immutable `sha-<commit>` tag and a `latest` tag.
4. The workflow updates `sendnoti-gitops/be/values.yaml` with the image repository and immutable tag, then commits that change to `main`.
5. Argo CD detects the Git change and syncs the Helm chart at `sendnoti-gitops/be`.

The workflow uses the same repository for application source and GitOps manifests. No separate `sendnoti-gitops` repository or `GITOPS_TOKEN` is required. Its GitOps commit does not trigger another image build because the workflow only watches backend source and its own workflow file.

## Initial setup

1. Create/use one GitHub repository named `sendnoti`.
2. Push this directory's contents to the repository's `main` branch.
3. In GitHub, allow Actions to use a `GITHUB_TOKEN` with read/write repository contents and package write permissions. Branch protection must permit the workflow bot to push the GitOps tag update, or that rule must be adjusted.
4. After the first successful workflow run, confirm the GHCR package `sendnoti-be` exists.
5. If the GHCR package is private, create an imagePullSecret in Kubernetes namespace `boat` and configure `image.pullSecrets` in `sendnoti-gitops/be/values.yaml`.
6. Replace the GitHub owner placeholder in `sendnoti-gitops/argocd/sendnoti-be.yaml`, then apply that Application manifest or create the equivalent Argo CD Application in the UI.

Do not sync the Helm chart while `image.tag` is `pending`; wait for the first successful image publication and tag update.

## Production prerequisites

- Configure Vault key `apps/sendnoti-be-30090` with the required values from `sendnoti-be/.env.example`, especially `MSTEAM_WEBHOOK_URL` / project-specific Teams webhook values and `TRUSTED_NETWORKS`.
- Confirm `vault-backend` is a `ClusterSecretStore`; adjust the ExternalSecret kind if it is a namespaced `SecretStore`.
- Confirm NodePort 30090 is available.
- Confirm the network policy/firewall permits trusted cbevent clients to reach the service. The application checks the socket peer IP and does not trust `X-Forwarded-For`.

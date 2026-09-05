# Configuring GitHub Container Registry with Kubernetes for Private Images

- Canonical URL: https://imzihad21.github.io/articles/a/configuring-github-container-registry-with-kubernetes-for-private-images-1pm0/
- Source URL: https://dev.to/imzihad21/configuring-github-container-registry-with-kubernetes-for-private-images-1pm0
- Web View: https://imzihad21.github.io/articles/a/configuring-github-container-registry-with-kubernetes-for-private-images-1pm0/
- Published: 2026-05-14T15:36:42.000Z
- Modified: 2026-05-14T15:36:42.000Z
- Reading time: 5 minutes
- Tags: devops, kubernetes, docker, ghcr

Private container images are common in real production workloads. Your app image might live in GitHub Container Registry, but Kubernetes cannot pull that image unless the cluster has valid registry credentials.

This guide shows a YAML-first setup for pulling private images from GitHub Container Registry in Kubernetes.

GitHub Container Registry uses `ghcr.io`, and private image pulls require authentication. GitHub documents personal access token authentication for Container Registry, and private package access usually needs at least `read:packages`. ([GitHub Docs][1])

### Why It Matters

* Keep application images private.
* Deploy private images directly from `ghcr.io`.
* Avoid manual login on cluster nodes.
* Keep workload YAML clean with ServiceAccounts.
* Fits well with GitOps, CI/CD, Helm, and Kustomize.

### Core Concepts

#### 1. GitHub Container Registry Image

A private GHCR image usually looks like this:

```text
ghcr.io/OWNER/IMAGE_NAME:TAG
```

Example:

```text
ghcr.io/my-org/private-api:1.0.0
```

Kubernetes will try to pull this image when the Pod starts. If the image is private and no credentials are configured, the Pod will fail with errors like:

```text
ImagePullBackOff
ErrImagePull
unauthorized
```

#### 2. Registry Credential Secret

Kubernetes uses an image pull Secret to authenticate with a private registry. The standard Secret type is:

```text
kubernetes.io/dockerconfigjson
```

Kubernetes supports using a Secret with private registry credentials and referencing it through `imagePullSecrets` in a Pod spec. ([Kubernetes][2])

The Docker config JSON looks like this before base64 encoding:

```json
{
  "auths": {
    "ghcr.io": {
      "username": "YOUR_GITHUB_USERNAME",
      "password": "YOUR_GITHUB_TOKEN",
      "email": "you@example.com",
      "auth": "BASE64_USERNAME_COLON_TOKEN"
    }
  }
}
```

The `auth` value is:

```text
base64(YOUR_GITHUB_USERNAME:YOUR_GITHUB_TOKEN)
```

Then the whole JSON content is base64 encoded and placed in the Kubernetes Secret.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ghcr-secret
  namespace: demo
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: BASE64_DOCKER_CONFIG_JSON
```

This Secret must exist in the same namespace as the Pod that uses it.

#### 3. ServiceAccount-Based Image Pull

Instead of adding `imagePullSecrets` to every Deployment, attach it to a ServiceAccount.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: demo
imagePullSecrets:
  - name: ghcr-secret
```

A ServiceAccount gives an identity to Pods, and Pods can be configured to use a specific ServiceAccount. Kubernetes documents ServiceAccounts as API objects used by processes running inside Pods. ([Kubernetes][3])

This keeps the Deployment clean.

#### 4. Deployment Using the ServiceAccount

Now the Deployment only needs to reference the ServiceAccount.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-api
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: private-api
  template:
    metadata:
      labels:
        app: private-api
    spec:
      serviceAccountName: app-service-account
      containers:
        - name: private-api
          image: ghcr.io/my-org/private-api:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
```

At startup, Kubernetes uses the ServiceAccount’s `imagePullSecrets`, reads the GHCR credentials, and pulls the private image. Clean Deployment YAML, no repeated registry config drama.

#### 5. Complete YAML Setup

A minimal full setup can be kept in one file:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: demo

---
apiVersion: v1
kind: Secret
metadata:
  name: ghcr-secret
  namespace: demo
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: BASE64_DOCKER_CONFIG_JSON

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: app-service-account
  namespace: demo
imagePullSecrets:
  - name: ghcr-secret

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-api
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: private-api
  template:
    metadata:
      labels:
        app: private-api
    spec:
      serviceAccountName: app-service-account
      containers:
        - name: private-api
          image: ghcr.io/my-org/private-api:1.0.0
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
```

### Practical Example

Suppose your private image is:

```text
ghcr.io/octocat/order-service:2026.05.14
```

Your Kubernetes objects should look like this:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production

---
apiVersion: v1
kind: Secret
metadata:
  name: ghcr-secret
  namespace: production
type: kubernetes.io/dockerconfigjson
data:
  .dockerconfigjson: BASE64_DOCKER_CONFIG_JSON

---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: order-service-sa
  namespace: production
imagePullSecrets:
  - name: ghcr-secret

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
  namespace: production
spec:
  replicas: 2
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      serviceAccountName: order-service-sa
      containers:
        - name: order-service
          image: ghcr.io/octocat/order-service:2026.05.14
          imagePullPolicy: IfNotPresent
          ports:
            - containerPort: 8080
```

Same cluster, private image, YAML-managed deployment, zero manual registry login.

### Alternative: Direct `imagePullSecrets`

For small apps, you can reference the Secret directly from the Deployment.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: private-api
  namespace: demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: private-api
  template:
    metadata:
      labels:
        app: private-api
    spec:
      imagePullSecrets:
        - name: ghcr-secret
      containers:
        - name: private-api
          image: ghcr.io/my-org/private-api:1.0.0
```

This works, but it gets repetitive when many workloads need private images.

The ServiceAccount approach is usually cleaner.

### GitOps-Friendly Secret Handling

Raw Kubernetes Secrets are only base64 encoded. They are not encrypted.

Avoid committing this directly to Git:

```yaml
data:
  .dockerconfigjson: BASE64_DOCKER_CONFIG_JSON
```

Better production options:

* Sealed Secrets
* External Secrets Operator
* SOPS with age or KMS
* Vault
* AWS Secrets Manager
* Google Secret Manager
* Azure Key Vault

A better flow is:

```text
Secret Manager
  -> ExternalSecret
  -> Kubernetes Secret
  -> ServiceAccount
  -> Deployment
```

Example `ExternalSecret` shape:

```yaml
apiVersion: external-secrets.io/v1
kind: ExternalSecret
metadata:
  name: ghcr-secret
  namespace: demo
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: cluster-secret-store
    kind: ClusterSecretStore
  target:
    name: ghcr-secret
    creationPolicy: Owner
    template:
      type: kubernetes.io/dockerconfigjson
      data:
        .dockerconfigjson: |
          {
            "auths": {
              "ghcr.io": {
                "username": "{{ .githubUsername }}",
                "password": "{{ .githubToken }}",
                "email": "you@example.com",
                "auth": "{{ printf "%s:%s" .githubUsername .githubToken | b64enc }}"
              }
            }
          }
  data:
    - secretKey: githubUsername
      remoteRef:
        key: github-username
    - secretKey: githubToken
      remoteRef:
        key: github-token
```

### Common Mistakes

* Creating the Secret in the wrong namespace.
* Using a GitHub token without `read:packages`.
* Forgetting to give the token access to the private package or repository.
* Committing raw base64 Kubernetes Secrets to Git.
* Adding `imagePullSecrets` to the Deployment but using a different ServiceAccount setup.
* Typing the image path incorrectly, especially the owner or package name.
* Using `latest` everywhere and not knowing what version is running.

### Quick Recap

* GHCR private images need registry authentication.
* Kubernetes uses `kubernetes.io/dockerconfigjson` Secrets for private registry pulls.
* Attach the Secret to a ServiceAccount for clean workload YAML.
* Use `serviceAccountName` in the Deployment.
* Store real credentials outside Git when possible.
* Prefer pinned image tags over `latest`.

### Next Steps

1. Move the GHCR token into External Secrets or Sealed Secrets.
2. Create one ServiceAccount per application or namespace.
3. Add image tag promotion through CI/CD.
4. Rotate GitHub package tokens periodically.
5. Add deployment checks for `ImagePullBackOff` and failed image pulls.

[1]: https://docs.github.com/packages/working-with-a-github-packages-registry/working-with-the-container-registry "Working with the Container registry"
[2]: https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/ "Pull an Image from a Private Registry"
[3]: https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/ "Configure Service Accounts for Pods"
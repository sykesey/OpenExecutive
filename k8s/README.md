# Kubernetes deployment

These manifests deploy the API and UI images published by
`.github/workflows/publish-images.yml` into the `openexecutive` namespace.
They intentionally do not include a secret manifest: credentials must not be
committed to the repository.

## Prerequisites

1. Publish the images through GitHub Actions and replace the `latest` tags in
   `kustomization.yaml` with the desired immutable `sha-<commit>` tags.
2. If the GHCR packages are private, create a GitHub token with `read:packages`
   and then create the pull secret (use a GitHub user that can read the package):

   ```sh
   kubectl create namespace openexecutive --dry-run=client -o yaml | kubectl apply -f -
   kubectl -n openexecutive create secret docker-registry ghcr-pull \
     --docker-server=ghcr.io \
     --docker-username=GITHUB_USERNAME \
     --docker-password=GITHUB_TOKEN
   ```

3. Create separate API and UI runtime secrets. `BACKEND_SHARED_SECRET` must be
   the same random value in both. The API is configured to use OpenRouter and
   needs `OPENROUTER_API_KEY`, `BACKEND_SHARED_SECRET`, and
   `EXEC_EMAIL_ADDRESS`; `ANTHROPIC_API_KEY` is optional for an overlay that
   disables OpenRouter. The UI needs `AUTH_SECRET`, `AUTH_GOOGLE_ID`,
   `AUTH_GOOGLE_SECRET`, `AUTH_URL`, `ALLOWED_EMAILS`, and
   `BACKEND_SHARED_SECRET`. Use an external-secrets controller in production,
   or create the native Secrets through your secret-management process.
4. Replace `BACKEND_ALLOWED_ORIGINS` in `configmap.yaml` with the public UI
   origin. Configure that same origin as `AUTH_URL` and as the Google OAuth
   callback origin: `<origin>/api/auth/callback/google`.

The base ConfigMap pins the current Council defaults: `claude-sonnet-5` for
general and research calls, `claude-opus-5` for deep-reasoning specialists,
and `claude-haiku-4-5-20251001` for lightweight routing. Override these in an
environment-specific ConfigMap overlay only when you intentionally want a
different supported model tier.

## Apply and verify

```sh
kubectl apply -k k8s/
kubectl -n openexecutive rollout status deployment/openexecutive-api --timeout=10m
kubectl -n openexecutive rollout status deployment/openexecutive-ui --timeout=5m
kubectl -n openexecutive get pods,services,pvc
```

The API intentionally has one replica and uses a `Recreate` update strategy:
it owns the scheduler and its `/data` volume is `ReadWriteOnce`. The UI has two
stateless replicas and reaches the API only through `openexecutive-api` inside
the cluster. The API Service is therefore never publicly exposed.

To publish the UI, adapt `ingress.example.yaml` to the cluster's ingress class,
hostname, and TLS configuration, then apply it separately.

## Optional self-hosted Honcho

Honcho comprises Postgres with pgvector, an API, a background deriver, and a
local embedding sidecar. It is intentionally disabled in the base
kustomization. The live cluster currently has only 2 CPUs and about 3.8 GiB
allocatable memory; it cannot safely host the existing API/UI and this stack.
Provision at least 4 CPUs and 8 GiB allocatable memory (or place Honcho on a
separate node) before enabling it.

After the new `openexecutive-honcho` GHCR image has been published, pin its
`sha-<commit>` tag in `kustomization.yaml`. Create these Secrets through your
secret manager before applying the overlay:

| Secret | Required keys |
|---|---|
| `openexecutive-honcho-postgres-secrets` | `POSTGRES_DB`, `POSTGRES_USER`, `POSTGRES_PASSWORD` |
| `openexecutive-honcho-secrets` | `DB_CONNECTION_URI`, `LLM_OPENAI_API_KEY`, `AUTH_JWT_SECRET`, `EMBEDDING_MODEL_CONFIG__OVERRIDES__BASE_URL`, `EMBEDDING_MODEL_CONFIG__OVERRIDES__API_KEY`, `EMBEDDING_VECTOR_DIMENSIONS`, `DERIVER_WORKERS` |

Set `DB_CONNECTION_URI` to
`postgresql+psycopg://<user>:<password>@openexecutive-honcho-postgres:5432/<database>`.
Set `LLM_OPENAI_API_KEY` to the OpenRouter key, the embedding base URL to
`http://127.0.0.1:8001/v1`, its API key to `local-no-auth-needed`, vector
dimensions to `384`, and `DERIVER_WORKERS` to `2`.

Apply the optional workload only after those prerequisites:

```sh
kubectl apply -k k8s/
kubectl apply -f k8s/honcho-enable.yaml
kubectl -n openexecutive rollout status statefulset/openexecutive-honcho-postgres --timeout=10m
kubectl -n openexecutive rollout status deployment/openexecutive-honcho --timeout=10m
```

Then mint a workspace-scoped Honcho token, place it in
`openexecutive-api-secrets` as `HONCHO_API_KEY`, and enable
`HONCHO_ENABLED=true` through an environment-specific ConfigMap overlay. The
base configuration keeps the integration off until that key exists so the API
never crash-loops during initial Honcho provisioning.

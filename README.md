# mlflow-platform

Kubernetes manifests for running an MLflow server with PostgreSQL for the
backend store and MinIO as the artifact store. Manifests are organized with
Kustomize to support local overlays.

## Layout
- `base/` — core resources (namespace, PostgreSQL, MinIO, MLflow, ingress).
  - `base/mlflow/` — MLflow deployment, service, ingress, config map.
  - `base/postgres/` — statefulset, service, PVC, credentials secret.
  - `base/minio/` — statefulset, service, PVC, credentials secret.
- `overlays/local/` — uses the base plus a job to create the `mlflow` bucket in
  MinIO.

## Prerequisites
- Kubernetes cluster and `kustomize` installed.
- Ingress controller (e.g., NGINX) for `mlflow.local`.
- (Optional) `/etc/hosts` entry: `127.0.0.1 mlflow.local` if you access via a
  local ingress.

## Deploy (local overlay) with kustomize
```sh
kustomize build overlays/local | kubectl apply -f -
```

What this does:
- Creates namespace `mlflow`.
- Deploys PostgreSQL (`postgres` service on 5432) with a 5Gi PVC.
- Deploys MinIO (`minio` service on 9000 API / 9001 console) with a 5Gi PVC and
  `minio-creds` secret.
- Runs a one-time job to ensure the `mlflow` bucket exists.
- Deploys MLflow (port 5000 exposed via `mlflow` service on 80 and ingress
  `mlflow.local`) configured to use PostgreSQL and MinIO (`s3://mlflow`).

## Access
- MLflow UI:
  - Via ingress: http://mlflow.local/ (requires ingress and DNS/hosts entry).
  - Via port-forward (no ingress needed):
    ```sh
    kubectl port-forward svc/mlflow 5000:80 -n mlflow
    ```
    Then open http://localhost:5000/.
- MinIO API/Console: cluster-internal service `minio:9000/9001`; port-forward
  if needed:
  ```sh
  kubectl port-forward svc/minio 9000:9000 9001:9001 -n mlflow
  ```

## GitOps / Argo CD
- The file `mlflow-application.yaml` contains an Argo CD `Application` manifest
  that can be used to manage this stack via GitOps.
- Apply the file to argocd namespace to continuously
  sync the MLflow + PostgreSQL + MinIO setup into your cluster.

## Configuration points
- Credentials:
  - PostgreSQL: `base/postgres/secret.yaml` (`postgres-secret`).
  - MinIO: `base/minio/secret.yaml` (`minio-creds`, used by MLflow and bucket job).
- Storage sizes:
  - PostgreSQL PVC: `base/postgres/pvc.yaml` (default 5Gi).
  - MinIO PVC: `base/minio/pvc.yaml` (default 5Gi).
- MLflow artifact config: `base/mlflow/configmap.yaml`
  (`MLFLOW_ARTIFACT_ROOT=s3://mlflow`,
  `MLFLOW_S3_ENDPOINT_URL=http://minio.mlflow.svc.cluster.local:9000`).
- Ingress host: `base/mlflow/ingress.yaml` (`mlflow.local`).
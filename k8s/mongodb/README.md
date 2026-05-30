# MongoDB (Community Operator) — Phase 0

Single-node MongoDB **replica set** (`replSet=mongodb`) on k3s, managed by the MongoDB
Community Operator. The replica set is required so the Wikipedia pipeline can use **change
streams**. Internal-only (ClusterIP); reach it from a laptop via `kubectl port-forward`.

See the design: Obsidian `Projects/wikipedia-edit-stream/`.

## Layout

- `../mongodb-operator/app.yaml` — ArgoCD app: installs the operator + CRDs (Helm chart
  `community-operator`), watching the `mongodb` namespace.
- `app.yaml` — ArgoCD app for this directory (the CR below).
- `namespace.yaml` — the `mongodb` namespace.
- `mongodbcommunity.yaml` — the `MongoDBCommunity` resource (1 member, SCRAM auth, a
  `wikipedia` app user, 10Gi data volume on `local-path`).

## One-time bootstrap: create the password Secret (NOT in git)

The CR references Secret `wikipedia-db-password`. It is deliberately kept out of GitOps so
`selfHeal` can't overwrite it and so a credential never lands in the repo. Create it once:

```bash
ssh runnyeye
kubectl create namespace mongodb --dry-run=client -o yaml | kubectl apply -f -
kubectl -n mongodb create secret generic wikipedia-db-password \
  --from-literal=password="$(openssl rand -base64 24)"
```

(Future: replace this manual step with Sealed Secrets / SOPS for a fully GitOps-native flow.)

## Deploy

Commit + push the new files; ArgoCD picks them up via the root app-of-apps. The
`mongodb-operator` app must become healthy before the `mongodb` app can reconcile the CR —
ArgoCD retries automatically (`SkipDryRunOnMissingResource` is set on the CR for the first
pass).

## Verify

```bash
# Operator + CR healthy, pod running:
kubectl -n mongodb get pods
kubectl -n mongodb get mongodbcommunity mongodb -o jsonpath='{.status.phase}'   # -> Running

# Replica set initiated (operator does rs.initiate automatically):
kubectl -n mongodb exec -it mongodb-0 -c mongod -- mongosh --quiet \
  -u wikipedia -p "$(kubectl -n mongodb get secret wikipedia-db-password -o jsonpath='{.data.password}' | base64 -d)" \
  --authenticationDatabase admin --eval 'rs.status().ok'                         # -> 1

# Change streams work on a single-member RS:
kubectl -n mongodb exec -it mongodb-0 -c mongod -- mongosh --quiet \
  -u wikipedia -p "<password>" --authenticationDatabase admin wikipedia \
  --eval 'const cs = db.smoke.watch(); db.smoke.insertOne({hi:1}); printjson(cs.tryNext());'
```

### Connect with Compass (laptop)

```bash
kubectl -n mongodb port-forward svc/mongodb-svc 27017:27017
```

Then connect Compass to:

```
mongodb://wikipedia:<password>@localhost:27017/?authSource=admin&directConnection=true
```

`directConnection=true` is required: it stops the driver from doing replica-set topology
discovery, which would otherwise try to reach the pod's internal DNS name (unreachable from
the laptop). Get the full string the operator generated with:

```bash
kubectl -n mongodb get secret wikipedia-mongodb-connection \
  -o jsonpath='{.data.connectionString\.standard}' | base64 -d
```

## In-cluster connection (for later phases)

Pipeline services use:

```
mongodb://wikipedia:<password>@mongodb-svc.mongodb.svc.cluster.local:27017/wikipedia?replicaSet=mongodb&authSource=admin
```

The operator publishes this in the `wikipedia-mongodb-connection` Secret (namespace
`mongodb`). Apps in another namespace will need that value copied/synced into their
namespace — handled when those services are built.

## Data safety / teardown

Data lives in PVCs created by the StatefulSet controller (`data-volume-mongodb-0`), which
ArgoCD does **not** manage and does not prune. Deleting the `mongodb` app or removing the
CR deletes the StatefulSet but **retains** the PVCs, so data survives an app teardown and is
reused if the CR is recreated. Only an explicit `kubectl delete pvc` removes the data.

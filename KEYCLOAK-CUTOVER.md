# Keycloak ownership handoff

Keep the `teachme` namespace and every Keycloak resource name. This cutover
changes Argo CD ownership, not the database, ingress hostname, or image. Keep
the two Keycloak ExternalSecrets under `external-secrets-config`.

## 1. Prepare and restore the route

Before deploying either PR, use an authenticated production context to
record the Keycloak claim UID and bound PV name, confirm they are the original
storage, and verify a current restorable PostgreSQL backup on separate storage.
If the claim is missing, Pending, or bound to replacement storage, stop and
investigate the original PV/backup; do not initialize an empty database.

Deploy the preparation PR by itself. It disables Teachly pruning and
cascading deletion, scales the Teachly frontend, API and database to zero,
and puts Keycloak's ingress ahead of the unused Teachly claims. The claims
stay declared; a `WaitForFirstConsumer` claim may still be Pending while
Teachly is off. Do not create a consumer just to make Teachly show Healthy.

Use an authenticated production Kubernetes/Argo CD context:

```sh
kubectl -n argocd get application teachme -o json | jq '{
  prune: .spec.syncPolicy.automated.prune,
  finalizers: .metadata.finalizers,
  operation: .operation,
  revision: .status.operationState.syncResult.revision,
  phase: .status.operationState.phase
}'
kubectl -n teachme get pvc -o custom-columns=NAME:.metadata.name,UID:.metadata.uid,STATUS:.status.phase,VOLUME:.spec.volumeName,CLASS:.spec.storageClassName
kubectl -n teachme get deployment,ingress,endpointslice,externalsecret
```

If Teachly is still running the **old** sync (waiting at the assets claim),
terminate that operation in Argo CD and sync the preparation revision with
prune unchecked. Before proceeding, confirm no old prune-enabled operation
is in flight, live Teachly has `prune: false` and no resources finalizer, and
the live namespace and Keycloak claim have `Prune=false,Delete=false`.

Confirm the claim UID and PV name still match. Verify Keycloak has ready
endpoints and the expected public issuers:

```sh
set -e
for realm in personal schreinerei teachly; do
  issuer="https://auth.jakob-lingel.dev/realms/$realm"
  curl -fsS --max-time 15 "$issuer/.well-known/openid-configuration" |
    jq -e --arg issuer "$issuer" '.issuer == $issuer'
done
```

## 2. Transfer ownership after the checks above pass

Deploy the extraction PR separately. Do not manually prune Teachly or delete
its namespace. Argo CD's new `keycloak` Application adopts the existing
objects in place. Leave the Keycloak image workflow idle during the handoff.

Check that `keycloak` is Synced/Healthy at the extraction revision, has no
shared-resource warning after Teachly refreshes, and that the original
Keycloak claim UID and PV name are unchanged. Recheck ready endpoints and
the issuers above, then perform a real sign-in through a dependent service.
Teachly should still have zero replicas for all three deployments.

If adoption fails, revert **only the extraction change**, keeping step 1's
prune protection. Confirm Teachly adopts the original resources again and
recheck the issuer and claim. Do not delete resources to undo the handoff.

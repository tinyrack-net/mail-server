```bash
kubectl create namespace sealed-secrets
kubectl -n sealed-secrets apply -f ./flux/main.key.yaml --force
```

```bash
flux bootstrap github \
  --owner=tinyrack94 \
  --repository=public \
  --branch=main \
  --path=./flux/clusters/production \
  --owner=tinyrack-net
```

```bash
kubectl create secret generic docmost-secret \
        --namespace docmost-system \
        --dry-run=client \
        --from-literal=SOME_SECRET_KEY=SOME_SECRET_VALUE \
        --from-literal=SOME_SECRET_KEY=SOME_SECRET_VALUE \
        --from-literal=SOME_SECRET_KEY=SOME_SECRET_VALUE -o yaml \
        | kubeseal --cert ./tinyrack-production-key.crt \
        > ./some.secret.yaml
```
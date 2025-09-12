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